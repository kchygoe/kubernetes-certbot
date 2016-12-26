# kubernetes-certbot

Uses [certbot][certbot] to obtain an X.509 certificate from [Let's encrypt][letsencrypt] and stores it as secret in
[Kubernetes][kubernetes].

## Usage

Create a service:

```yml
# kubernetes-certbot-svc.yml
apiVersion: v1
kind: Service
metadata:
  name: kubernetes-certbot
spec:
  selector:
    name: kubernetes-certbot
  ports:
    - name: http
      port: 80
```

Create a replication controller:

```yml
# kubernetes-certbot-rc.yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubernetes-certbot
spec:
  replicas: 1
  template:
    metadata:
      labels:
        name: kubernetes-certbot
    spec:
      containers:
      - name: kubernetes-certbot
        image: choffmeister/kubernetes-certbot:latest
        imagePullPolicy: Always
        env:
          - name: SECRET_NAMESPACE
            value: default
          - name: SECRET_NAME_PREFIX
            value: foobar
        volumeMounts:
        - mountPath: /etc/letsencrypt
          name: letsencrypt-data
      volumes:
      - name: letsencrypt-data
        emptyDir: {}
```

Configure your front gateway (in this example [nginx][nginx]) to forward all incoming traffic for certbot to the service
you just created (this assumes, you have [kube-dns][kubedns] running, so that nginx is able to resolve the host
`kubernetes-certbot`):

```
# nginx.conf
server {
  listen 80 default_server;
  server_name _;

  location /.well-known/acme-challenge/ {
    proxy_pass http://kubernetes-certbot;
  }
}
```

Then, whenever you need a certificate, find out the name of the pod (let it be `${LETSENCRYPT_POD}` here) and execute:

```bash
kubectl exec -it ${LETSENCRYPT_POD} -- bash ./run.sh "secret-name" "mail@mydomain.com" "mydomain.com,www.mydomain.com" "namespace"
```

This will create a secret `foobar-secret-name` in the namespace `default` containing four entries for the individual
`.pem` files genereted by certbot.

[letsencrypt]: https://letsencrypt.org/
[certbot]: https://github.com/certbot/certbot
[kubernetes]: http://kubernetes.io/
[nginx]: https://nginx.org/
[kubedns]: https://github.com/kubernetes/kubernetes/tree/master/build/kube-dns
