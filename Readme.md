# Notion Doc  
https://www.notion.so/Implantando-com-eksctl-c57caf70428344f8bf0bdd8c10309ec6  


## Passos

1. Antes de iniciar a instalação é necessário:
- Ter as chaves de acesso ao CLI
- Permissões abaixo:
    - 
    
    ![Permissoes](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/6703ce07-ed68-472a-8825-37409c711658/Untitled.png)
    
- Instalar o eksctl
- Instalar e configurar o AWS CLI
    - Configurar com comando e informar suas chaves e region eu-central-1:
    - Se atente na region, pois tem region que não funciona o EKS.
    
    ```bash
    aws configure
    ```
    

1. Instalar seu cluster, modelo de configuração:
    1. cluster.yaml
    2. mais modelos em: [eksctl/examples at main · weaveworks/eksctl (github.com)](https://github.com/weaveworks/eksctl/tree/main/examples)
    
    ```bash
    apiVersion: eksctl.io/v1alpha5
    kind: ClusterConfig
    
    metadata:
      name: claroclustertest
      region: eu-central-1
    
    nodeGroups:
      - name: ng-1
        instanceType: t2.medium
        desiredCapacity: 1
        volumeSize: 20
      - name: ng-2
        instanceType: t2.medium
        desiredCapacity: 1
        volumeSize: 20
    
    cloudWatch:
        clusterLogging:
            # enable specific types of cluster control plane logs
            enableTypes: ["audit", "authenticator", "controllerManager"]
            # all supported types: "api", "audit", "authenticator", "controllerManager", "scheduler"
            # supported special values: "*" and "all"
    ```
    
2. Caso ainda não tenha registrado seu **dns** ou **subdomínio**, recomendo seguir o passo para fazer a criação do **hostedzone** via cli logo abaixo:
    1. criar um subdomain:
        1. Detalhe importante aqui, para seu **subdomain funcionar,** você precisa adicionar os NS da criação referente ao comando abaixo no seu  hosted principal, nesse caso, seria adicionar uma entrada do tipo NS em [**site.com.br](http://site.com.br) .** Os valores para NS do subdomain serão listados após sua criação ou estarão listados route53.
        
        ```bash
        aws route53 create-hosted-zone --name "subdomain.site.com.br." --caller-reference "external-dns-test-$(date +%s)"
        ```
        
        ![DNS](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/dafbb0b4-f092-4465-a2e4-9e9067e7b877/Untitled.png)
        
3. Criar IAM Policy com o json abaixo:
    1. Ir em IAM, Policy, Create policy:
        1. 
        
        ![Policy](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/da527b3d-a95d-446b-8107-6ea065945335/Untitled.png)

        ```bash
        {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Action": [
                        "route53:ChangeResourceRecordSets"
                    ],
                    "Resource": [
                        "arn:aws:route53:::hostedzone/*"
                    ]
                },
                {
                    "Effect": "Allow",
                    "Action": [
                        "route53:ListHostedZones",
                        "route53:ListResourceRecordSets"
                    ],
                    "Resource": [
                        "*"
                    ]
                }
            ]
        }
        ```
        
4. Criar OIDC para o seu cluster:
    1. use eksctl
    
    ```bash
    eksctl utils associate-iam-oidc-provider --region=eu-central-1 --cluster=NAME_YOUR_CLUSTER --approve
    ```
    
5. Criar serviceaccount
    1. use eksctl
    
    ```bash
    #Observar o nome do cluster e use --override-existing-serviceaccounts caso 
    #já exista um serviceaccount para seu cluster
    eksctl create iamserviceaccount --override-existing-serviceaccounts --name external-dns --namespace default --cluster NAME_YOUR_CLUSTER --attach-policy-arn ARN_POLICY_STEP_4 --region=eu-central-1 --approve
    ```
    
6. Criar external-dns para apontamento do **DNS** aos seus deployments.
    1. dns-external.yaml
    
    ```bash
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: external-dns
      # If you're using Amazon EKS with IAM Roles for Service Accounts, specify the following annotation.
      # Otherwise, you may safely omit it.
      annotations:
        # Substitute your account ID and IAM service role name below.
        #Procure pela sua role no IAM, terá o nome do seu cluster
        eks.amazonaws.com/role-arn: ARN_ROLE_YOUR_CLUSTER_IAM
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: external-dns
    rules:
    - apiGroups: [""]
      resources: ["services","endpoints","pods"]
      verbs: ["get","watch","list"]
    - apiGroups: ["extensions","networking.k8s.io"]
      resources: ["ingresses"]
      verbs: ["get","watch","list"]
    - apiGroups: [""]
      resources: ["nodes"]
      verbs: ["list","watch"]
    ---
    apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: external-dns-viewer
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: external-dns
    subjects:
    - kind: ServiceAccount
      name: external-dns
      namespace: default
    ---
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: external-dns
    spec:
      strategy:
        type: Recreate
      selector:
        matchLabels:
          app: external-dns
      template:
        metadata:
          labels:
            app: external-dns
          # If you're using kiam or kube2iam, specify the following annotation.
          # Otherwise, you may safely omit it.
          annotations:
            iam.amazonaws.com/role: ARN_ROLE_YOUR_CLUSTER_IAM
        spec:
          serviceAccountName: external-dns
          containers:
          - name: external-dns
            image: k8s.gcr.io/external-dns/external-dns:v0.7.6
            args:
            - --source=service
            - --source=ingress
            - --domain-filter=subdomain.seusite.com.br # will make ExternalDNS see only the hosted zones matching provided domain, omit to process all available hosted zones
            - --provider=aws
            - --policy=upsert-only # would prevent ExternalDNS from deleting any records, omit to enable full synchronization
            - --aws-zone-type=public # only look at public hosted zones (valid values are public, private or no value for both)
            - --registry=txt
            - --txt-owner-id=ID_HOSTEDZONE_IN_USE #Recupera com o comando no passo 3.a ou no Route53 diretamente
          securityContext:
            fsGroup: 65534 # For ExternalDNS to be able to read Kubernetes and AWS token files
    ```
    
7. Testando seu Cluster
    1. deploy-nginx-service.yaml
        
        ```bash
        apiVersion: v1
        kind: Service
        metadata:
          name: mockfront
          annotations:
            external-dns.alpha.kubernetes.io/hostname: mockfront.eks.renatodesouza.com.br
            cert-manager.io/cluster-issuer: letsencrypt
        spec:
          type: LoadBalancer
          ports:
          - port: 80
            name: http
            targetPort: 80
          selector:
            app: mockfront
        ---
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          labels:
            app: mockfront
          name: mockfront
        spec:
          replicas: 2
          selector:
            matchLabels:
              app: mockfront
          template:
            metadata:
              labels:
                app: mockfront
            spec:
              containers:
              - image: nginx
                name: mockfront
                ports:
                - containerPort: 80
                  name: http
        ```
        
        b. Deve ser criado seu service e deploy, logo após, uns 5 min, deve ser criado seu subdomain, que você escolheu no seu service e estará online para você.
        
8. Configurando o lets-encrypt para gerar seus certificados HTTPs. (fazendo)
9. Configurando a criação de seus certificados HTTPS diretamente na AWS. (fazendo)