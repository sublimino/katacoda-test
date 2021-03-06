Signing your images allows you to cryptographically verify the content of that image. You can use tools such as Portieris, which we'll cover later in this tutorial, to check image signatures at deploy time, to ensure that you trust the code that's running in your production environments. Some tools, including Harbor and Docker, refer to this feature as Content Trust.

Notary is a tool that manages signatures. It implements The Update Framework (TUF), a specification for securing application deployment. Both Notary and TUF are "Incubating" CNCF projects.

Using Notary, you can digitally sign and then verify the content of your container images at deploy time via their digest. Notary is typically integrated alongside a Container Registry to provide this functionality. In this tutorial, we use Harbor as both the Container Registry and the Notary server.

## Signing images

When you enable Content Trust, Docker pushes trust information into Notary when you push your image.

1. Enable Docker Content Trust.

    ```bash
    export DOCKER_CONTENT_TRUST=1
    export DOCKER_CONTENT_TRUST_SERVER=https://127.0.0.1:30004
    ```

2. Push an image to Harbor. Your image will push as normal, but afterwards you'll be prompted to create passphrases for two signing keys:
    - Your root key, which is used to initialize repositories.
    - Your repository key, which is used to sign the image content.

    Make sure to set these passphrases to something that you remember, because you'll need to refer back to them. In this demo they can be the same thing, but in production it's safer to use different passphrases for each key.

    ```bash
    docker tag 127.0.0.1:30002/library/demo-api:secure 127.0.0.1:30002/library/demo-api:signed
    docker push 127.0.0.1:30002/library/signed-demo-api:latest
    ```

    **OSX**: If you get a "certificate signed by unknown authority" error, you need to add the Harbor CA to Docker's trusted certificates.

    ```bash
    mkdir -p ~/.docker/tls/127.0.0.1:30004
    cp harbor-ca.crt ~/.docker/tls/127.0.0.1:30004/ca.crt
    ```

3. Check your image signature.

    ```bash
    docker trust inspect --pretty 127.0.0.1:30002/library/signed-demo-api:latest
    ```

## Verifiable trust

You've signed the image already using the repository key, but this key doesn't prove your identity. The repository key is also unique to that image repository, so you'll create different keys for each image repository that you sign. Notary lets the repository key holder add other people's key pairs so that they can sign the image, and users can use those people's public key to verify their signatures.

1. Create a signing key called `portierisdemo`.

    ```bash
    docker trust key generate portierisdemo
    ```

    The private key goes in to your Docker Content Trust directory. The public key is saved to `portierisdemo.pub` in your working directory.

2. Add your key as a signer in Notary.

    ```bash
    docker trust signer add --key=portierisdemo.pub portierisdemo 127.0.0.1:30002/library/signed-demo-api
    ```

3. Let's look at the image that we pushed with content trust turned off, for comparison.

    ```bash
    docker trust inspect --pretty 127.0.0.1:30002/library/demo-api:secure
    ```

    This image wasn't signed when we pushed it, so we get a message saying that there's no trust information:

    ```bash
    No signatures or cannot access 127.0.0.1:30002/library/demo-api:secure
    ```

4. To see the trust files stored on disk, examine the local `~/.docker/trust` directory:

    ```bash
    tree ~/.docker/trust
    ```

    Notary caches signatures locally so that you don't need to go to the server each time. Cached signature data is stored in `~/.docker/trust/tuf` in folders representing the image name.

    Signing keys are stored in `~/.docker/trust/private`. If you've pulled trust information from a repository before, Notary caches the repository key and verifies that the same repository key is still being used. If the repository key reported by the server is different to the one that Notary saw before, the pull is rejected because the information in the server might have been changed by a third party.

5. Delete the deployment that you created earlier.

    ```bash
    kubectl delete deployment demo-api
    ```

6. Add an ImagePullSecret for Portieris to use to pull trust data from Harbor.

    ```bash
    k create secret docker-registry harbor --docker-username=admin --docker-password=kubecon1234 --docker-server=127.0.0.1:30002 --docker-email=a@b.com
    kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"harbor"}]}'
    ```

## Portieris

Portieris is a Kubernetes admission controller, open sourced by IBM. When you create a new workload in Kubernetes, Portieris modifies the image tag in your workload specification to refer to the most recent signed version of the image before the Kubernetes API Server distributes the workload to your cluster workers. Your cluster workers pull the signed version of the image instead of the most recently tagged version, even if they are different. When pods are re-scheduled by your cluster, the new workers also pull the same signed version, rather than the tagged version that may have changed in between creating the workload and re-scheduling the pod.

1. Install Portieris.

    1. Deploy Portieris.

        ```bash
        kubectl apply -f portieris.yaml
        ```

    1. Wait for Portieris to start. This might take a couple of minutes.

        ```bash
        kubectl get pods --watch
        ```

        Wait for there to be at least one `portieris` pod running:

        ```bash
        NAME                         READY   STATUS    RESTARTS   AGE
        portieris-84f9cb7746-bxxqm   1/1     Running   0          18s
        ```

    1. Set up the Mutating Admission Webhook for Portieris. This tells Kubernetes to ask Portieris if a given resource is OK to deploy.

        ```bash
        kubectl apply -f webhook.yaml
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

        Look at the `spec` section, and the `repositories` subsection inside it. One image is shown, with `name: "*"` and `policy: trust: enabled: true`. `*` is a wildcard character, so this policy matches any image.

        Change `enabled: true` to `enabled: false`.

        When the policy is not enabled, we don't apply any trust requirement. Essentially, Portieris doesn't do anything at this point. Let's prove that.

    3. Try to deploy an unsigned image to the cluster. We've created a deployment definition for you.

        ```bash
        kubectl apply -f demo-api.yaml
        ```

        Portieris doesn't do anything with the deployment, so it's allowed to be deployed.

    4. Let's enable trust enforcement for our image. Create a new element in the `repositories` list, after the `*`:

        ```yaml
        repositories:
            - name: "127.0.0.1:30002/library/*demo-api"
              policy:
                trust:
                  enabled: true
                  trustServer: "https://127.0.0.1:30004"
        ```

        Then save and close the file.

    5. Delete your deployment and then try to deploy your unsigned image again.

        ```bash
        kubectl delete deployment demo-api
        kubectl apply -f demo-api.yaml
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

    7. Try to deploy your signed image. Change the image in demo-api.yaml to our signed image: `127.0.0.1:30002/library/signed-demo-api`

        ```bash
        vi demo-api.yaml
        kubectl apply -f demo-api.yaml
        ```

        This deployment is allowed.

You have signed your image and configured Portieris to require image signatures. You have seen that when it enforces content trust, Portieris modifies the image name to the digest of the signed image.

## Using named signers with Portieris

Portieris can verify the signatures from named signers, and only allow the deployment of an image once it has a valid signature from each one.

1. Edit the ClusterImagePolicy to enforce your particular signer.

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
            - name: "127.0.0.1:30002/library/*demo-api"
            policy:
                trust:
                enabled: true
                trustServer: "https://127.0.0.1:30004"
                signerSecrets:
                - name: portierisdemo
        ```

2. Try to deploy your signed image again. This time, the deployment is rejected. Your image is signed, but not by the `portierisdemo` signer.

    ```text
    admission webhook "trust.hooks.securityenforcement.admission.cloud.ibm.com" denied the request: Deny, no signature found for role portierisdemo
    ```

3. Sign the image using your `portierisdemo` key. Docker automatically signs images using all the keys that you have, so running the sign command again adds a signature for your newly created key.

    ```bash
    docker trust sign 127.0.0.1:30002/library/signed-demo-api:latest
    ```

4. Try to deploy your signed image once more. This time, the deployment is allowed.

You have created a signing key and configured Portieris to require images in your namespace to be signed using it. You can use the same signing key in multiple repositories to require a common root of trust.
