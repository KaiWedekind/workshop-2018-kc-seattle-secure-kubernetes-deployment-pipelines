### Configure Harbor project to use Clair
Turn on image scanning for the default library project by heading to the ui and selecting the `library` project, clicking the `Configuration` tab. Select `Prevent vulnerable images from running` and set the threshold to high. This will prevent vulnerable images with one or more `high` vulnerability CVEs from getting pulled from this registry. Set the option to scan on push.

Now build the image to be scanned. Clone the demo-API https://github.com/lukebond/demo-api onto your local machine and check out the `kubecon` branch.

```bash
git clone https://github.com/lukebond/demo-api.git
cd demo-api
git checkout kubecon
```

Build and push into Harbor with

```bash
docker build . -t $MINIKUBE_IP:30003/library/demo-api:vulnerable
docker push $MINIKUBE_IP:30003/library/demo-api:vulnerable
```

In the Harbor UI, enter the library project and view the demo-api image. If the `vulnerability` column shows as `not scanned`, head to the `Configuration tab` in the left hand side menu, click `vulnerability` and check if there is a date stamp for `Database updated on`. If you see a notification icon, Harbor is still downloading its CVE database, which can take some time depending on bandwidth. Wait until you see the time stamp, kick off a scan and return to the pushed image. The vulnerability rating should be showing as `HIGH`, indicating that there is one or more strong vulnerabilities in the image.

### Deploy vulnerable image

Edit the setup/demo-api.yaml file in this repo by replacing `<Minikube_IP>` with the minikube IP you found in a previous step. Enter the workshop repo and attempt to deploy the image from the registry into your minikube cluster.

```bash
kubectl apply -f setup/demo-api.yaml
```

Check if it has deployed with

```bash
kubectl get po
```

If you have configured harbor correctly, you will notice that the pod has an `ErrImagePull` error. Investigate why with

```bash
kubectl describe po <pod name>
```

As expected, the event log shows that the image was denied based on its high vulnerability.


### Fix and deploy secure images

To fix this, we need to update the dockerfile with a secure base image.

In the Dockerfile, change the base image to `alpine:3.4`, rebuild and push

```bash
docker build . -t <Minikube_IP>:30003/library/demo-api:secure
docker push <Minikube_IP>:30003/library/demo-api:secure
```

Then change the deployment yaml `demo-api.yaml` to reference your new secure image, with the secure tag and redeploy.

```bash
kubectl apply -f setup/demo-api.yaml
```

Check if the pod has come up. Due to port limitations in Minikube, if your pod throws a `CrashLoopBackOff` error, you can delete the service with `kubectl delete service/demo-api`, wait for the pod to come up and then reapply the yaml above.
If you go to your browser and enter your cluster IP with the port defined in the yaml, `http://<Minikube_IP>:30009/` you will see the demo api has been deployed successfully.
