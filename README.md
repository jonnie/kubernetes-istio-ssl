# Setting up SSL on Istio within Kubernetes

Having SSL enabled for any site is expected these days. If you use Istio with Kubernetes it isn't that obvious how to get this right. The topic is well documented on the Istio website however I have found Istio is very unforgiving if you get it wrong.

## Create your certificates

Skip this step if you already have the necessary files (`bundle.pem`, `cert.pem`, `chain.pem` and `key.pem`). 

Most people will know of and used LetsEncrypt and it is possible to integrate the process into Kubernetes and Istio. Personally I have struggled with this and have found an alternative much easier and less error prone.

The service I recommend is sslforfree.com, it's easy to use and if you create an account it will keep track of your certificates so you can see when they expire. Head over there and enter the domain name which you want to enable SSL for. If you want to include multiple sub-domains you can do that as well, just by separating each with a space:

Single domain:

    example.com

Multiple sub-domains:

    sub.example.com example.com

Wildcard domain (include all sub-domains):

    *.example.com

Follow the rest of the steps and you should end up with 3 text boxes with your private key, certificate and CA bundle. Copy these into local files on your computer using the following syntax:

    example.com.key.pem     <- contents of first box
    example.com.cert.pem    <- contents of second box
    example.com.chain.pem   <- contents of third box

Keep these safe, you probably won't need them again after installing them but you want to ensure no one else gets their hands on them.

## Add the files to Kubernetes

Create an `Secret` within Kubernetes by running the following:

    kubectl create -n istio-system secret tls istio-ingressgateway-certs \
        --key /path/to/certs/example.com.key.pem \
        --cert /path/to/certs/example.com.cert.pem

Next, you'll need to create another `Secret` for the CA certificates:

    kubectl create -n istio-system secret generic istio-ingressgateway-ca-certs \
       --from-file=/path/to/certs/example.com.chain.pem

Note: the filename you use here (i.e. the non-directory part of the `--from-file`) will be the filename on the ingress pod and thus you must configute the `Gateway` properly. Just ensure you refer to the correct filename (more below).

## Configure Istio 

Finally, we need to configure the rest of Istio:

    apiVersion: networking.istio.io/v1alpha3
    kind: Gateway
    metadata:
      name: gateway
    spec:
      selector:
        istio: ingressgateway
        servers:
        - port:
            number: 443
            name: https
            protocol: HTTPS
          tls:
            mode: SIMPLE
            serverCertificate: /etc/istio/ingressgateway-certs/tls.crt
            privateKey: /etc/istio/ingressgateway-certs/tls.key      
            caCertificates: /etc/istio/ingressgateway-ca-certs/example.com.chain.pem
    hosts:
    - "*.example.com"

Then apply the change:

    kubectl apply -f gateway.yaml

If you use a `VirtualService` for your ingress then you simply just need to refer to this `Gateway` by name:

    apiVersion: networking.istio.io/v1alpha3
    kind: VirtualService
    metadata:
      name: app
    spec:
      hosts:
      - "app.example.com"
      gateways:
      - gateway
      http:
      - route:
        - destination:
            host: app.default.svc.cluster.local
            port: 
              number: 8000