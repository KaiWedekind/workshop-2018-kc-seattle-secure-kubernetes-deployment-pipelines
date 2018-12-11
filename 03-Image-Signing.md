# Image Signing

## Creating signatures

Signing your images allows you to cryptographically verify the content of that image. You can use tools such as Portieris, which we'll cover later in this tutorial, to check image signatures at deploy time, to ensure that you trust the code that's running in your production environments. Some tools, including Harbor and Docker, refer to this feature as Content Trust.

Notary is a tool that manages signatures. It implements The Update Framework (TUF), a specification for securing application deployment. Both Notary and TUF are CNCF projects.

Using Notary, you can digitally sign and then verify the content of your container images at deploy time via their digest. Notary is typically integrated alongside a Container Registry to provide this functionality. In this tutorial, we use Harbor as both the Container Registry and the Notary server.

1. Enable Docker Content Trust.
    ```bash
    export DOCKER_CONTENT_TRUST=1
    export DOCKER_CONTENT_TRUST_SERVER=https://$MINIKUBE_IP:30003
    ```

2. Push an image to Harbor. Your image will push as normal, but afterwards you'll be prompted to create passphrases for two signing keys:
    - Your root key, which is used to initialize repositories.
    - Your repository key, which is used to sign the image content.

    Make sure to set these passphrases to something that you remember, because you'll need to refer back to them. In this demo they can be the same thing, but in production it's safer to use different passphrases for each key.

    ```bash
    docker tag $MINIKUBE_IP:30003/library/demo-api:secure $MINIKUBE_IP:30003/library/demo-api:signed
    docker push $MINIKUBE_IP:30003/library/demo-api:signed
    ```

3. You can check your image signature by using `docker trust inspect <Minikube_IP>:30003/library/demo-api:signed`.

4. You've signed the image already using the repository key, but this key doesn't prove your identity. The repository key is also unique to that image repository, so you'll create different keys for each image repository that you sign. Notary lets the repository key holder add other people's key pairs so that they can sign the image.

    1. Create a signing key called `portierisdemo`.
        ```bash
        cd ~
        docker trust key generate portierisdemo
        ```
        The private key goes in to your Docker Content Trust directory. The public key is saved to `portierisdemo.pub` in your working directory.

    2. Add your key as a signer in Notary.
        ```bash
        docker trust signer add --key=portierisdemo.pub portierisdemo $MINIKUBE_IP:30003/library/demo-api
        ```

5. Let's look at the image that we pushed in part 1, for comparison.

    ```bash
    docker trust inspect $MINIKUBE_IP:30003/library/demo-api:secure
    ```

    This image wasn't signed when we pushed it, so we get a message saying that there's no trust information:

    ```bash
    []
    No signatures or cannot access <Minikube_IP>:30003/library/demo-api:secure
    ```

6. To see the trust files stored on disk, examine the local `~/.docker/trust` directory:
    ```bash
    tree ~/.docker/trust
    ```

    Notary caches signatures locally so that you don't need to go to the server each time. Cached signature data is stored in `~/.docker/trust/tuf` in folders representing the image name. Signing keys are stored in `~/.docker/trust/private`.

7. Let's also delete the deployment that you created earlier.
    ```bash
    kubectl delete deployment demo-api
    ```

## Portieris

Portieris is a Kubernetes admission controller, open sourced by IBM. It integrates Notary image signing into your deployment pipelines by telling the Kubernetes API server to check with it whenever a resource that results in an image being run is created. It does this via a Mutating Admission Webhook.

1. Install Portieris.

    1. Add self signing CA to Portieris deployemnt
        ```bash
        sed -i '' "s/ca.pem: <sed-me>/ca.pem: $(cat ca.crt | base64)/" setup/portieris.yaml
        ```

    2. Deploy Portieris.
        ```bash
        kubectl create ns ibm-system
        kubectl apply -f setup/portieris.yaml
        ```

    3. Wait for Portieris to start. This might take a couple of minutes.
        ```bash
        kubectl get pods -n ibm-system --watch
        ```

        Wait for there to be at least one `portieris` pod running:
        ```bash
        NAME                         READY   STATUS    RESTARTS   AGE
        portieris-84f9cb7746-bxxqm   1/1     Running   0          18s
        ```

    4. Set up the Mutating Admission Webhook for Portieris. This tells Kubernetes to ask Portieris if a given resource is OK to deploy.
        ```bash
        kubectl apply -f setup/webhook.yaml
        ```

    Portieris installs two custom Kubernetes resources for managing it: ImagePolicies and ClusterImagePolicies. If an ImagePolicy exists in the same Kubernetes namespace as your resources, Portieris uses that to decide what rules to enforce on your resources. Otherwise, Portieris uses the ClusterImagePolicy.

1. Out of the box, Portieris installs ImagePolicies into the `kube-system` and `ibm-system` namespaces, and a ClusterImagePolicy. Let's have a look at what's created.
    1. List your ClusterImagePolicies.
        ```bash
        kubectl get ClusterImagePolicies
        ```

        One ClusterImagePolicy is shown: `portieris-default-cluster-image-policy`.
    2. Edit the `portieris-default-cluster-image-policy` policy.
        ```bash
        kubectl edit ClusterImagePolicy portieris-default-cluster-image-policy
        ```

        Look at the `spec` section, and the `repositories` subsection inside it. One image is shown, with `name: "*"` and `policy: null`. `*` is a wildcard character, so this policy matches any image. When the policy is null, we don't apply any trust requirement. Essentially Portieris doesn't do anything at this point. Let's prove that quickly.

    3. Try to deploy an unsigned image to the cluster. We've created a deployment definition for you.

        ```bash
        kubectl apply -f setup/demo-api.yaml
        ```

        Portieris doesn't do anything with the deployment, so it's allowed to be deployed.

    4. Let's enable trust enforcement for our image. Create a new element in the `repositories` list, after the `*`:
        ```yaml
        repositories:
            - name: "<Minikube_IP>:30003/library/demo-api"
              policy:
                trust:
                  enabled: true
                  trustServer: "<Minikube_IP>:4443"
        ```

        Then save and close the file.

    5. Try to deploy your unsigned image again.

        ```bash
        kubectl apply -f setup/demo-api.yaml
        ```

        The deployment is rejected because it isn't signed.

        ```text
        admission webhook "trust.hooks.securityenforcement.admission.cloud.ibm.com" denied the request: Deny, failed to get content trust information: No valid trust data for secure
        ```

    6. Portieris doesn't prevent pods from restarting, even if the pod no longer satisfies your policy. This prevents you from getting an outage if your pods crash but they don't match your policy.

        Delete the pods in your deployment, and then watch as they are re-created.

        ```bash
        kubectl delete pods -l app=demo-api
        kubectl get pods -l app=demo-api --watch
        ```

    7. Try to deploy your signed image.

        ```bash
        kubectl apply -f mysigneddeploy.yaml
        ```

        This deployment is allowed.

    8. Edit the ClusterImagePolicy to enforce your particular signer.

        1. Create a secret for the public key.

            ```bash
            kubectl create secret generic portierisdemo --from-literal=name=portierisdemo --from-file=publicKey=portierisdemo.pub
            ```

        2. Edit the ClusterImagePolicy to enforce it.

            ```bash
            kubectl edit clusterimagepolicy portieris-default-cluster-image-policy
            ```

            Add the signer secret as a required key for our demo-api repository:

            ```yaml
            repositories:
                - name: "<Minikube_IP>:30003/library/demo-api"
                policy:
                    trust:
                    enabled: true
                    trustServer: "<Minikube_IP>:4443"
                    signerSecrets:
                    - name: portierisdemo
            ```

    9. Try to deploy your signed image again. This time, the deployment is rejected.

        ```text
        admission webhook "trust.hooks.securityenforcement.admission.cloud.ibm.com" denied the request: Deny, no signature found for role portierisdemo
        ```

    10. Sign the image using your `portierisdemo` key. Images are automatically signed using all the keys that you have, so running the sign command adds a signature for your newly created key.
        ```bash
        docker trust sign $MINIKUBE_IP:30003/library/demo-api:signed
        ```

    11. Try to deploy your signed image once more. This time, the deployment is allowed.
