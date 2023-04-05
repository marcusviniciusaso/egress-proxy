# egress-http-proxy

Esse proxy Egress tem por objetivo ser um "proxy de proxy", um proxy intermediário que encaminha as requisições para o proxy principal. Essa solução foi construída com a imagem registry.redhat.io/openshift4/ose-egress-http-proxy:v4.10.0-202204211158.p0.g5d526c4.assembly.stream do Egress.

## Deploy

Essa solução utiliza uma imagem oficial do Egress HTTP Proxy com algumas alterações no entrypoint para configurar o "proxy de proxy". Internamente, o Egress HTTP Proxy utiliza o Squid como motor para o proxy. Dessa forma, as configurações implementadas estão no padrão interpretado pelo Squid.

A instalação é feita por meio de um template que provisiona os recursos necessários para o Egress HTTP Proxy. Atenção aos valores dos parâmetros antes de executar o comando:

NAMESPACE: Nome do namespace que será instalado o Egress HTTP Proxy.<br>
APPLICATION_NAME: Nome do app (valor à escolha).<br>
SERVICE_PORT: Porta do serviço do Egress HTTP Proxy (valor à escolha).<br>
PROXY_HOST: Endereço do proxy principal.<br>
PROXY_PORT: Porta do proxy principal.<br>
PROXY_USER: Usuário do proxy principal.<br>
PROXY_PASSWORD: Senha do proxy principal.

```
oc new-app -f egress-http-template.yaml -p NAMESPACE=squid -p APPLICATION_NAME=egress-http-proxy -p SERVICE_PORT=8080 -p PROXY_HOST=10.217.5.61 -p PROXY_PORT=3128 -p PROXY_USER=admin -p PROXY_PASSWORD=admin
```

Esse comando criará um pod que, por falta de permissão, pode não estar rodando corretamente. Criará também um ConfigMap com as configurações do Squid, um Service para acesso ao Egress HTTP Proxy e um DeploymentConfig.

Caso o pod não esteja rodando corretamente, é necessário elevar a permissão do DeploymentConfig com o SCC anyuid:
```
oc create sa egress-sa
oc adm policy add-scc-to-user anyuid -z egress-sa
oc set sa deploymentConfig/egress-http-proxy egress-sa
```

Após isso o Pod deve estar rodando corretamente.

## Teste

Para realizar o teste se o Egress proxy está funcionando corretamente, abrir outro pod no namespace e executar:
```
curl http://www.google.com/ --proxy EGRESS_SERVICE_HOST:EGRESS_SERVICE_PORT | grep "<title>Google</title>"
```

O comando curl deve retornar resultado.

## Remoção (Cuidado, apagará o app e seus recursos!)

Caso seja necessário excluir todos os recursos criados pelo template, executar os comandos abaixo:

```
oc delete all -l app=egress-http-template
oc delete configmap -l app=egress-http-template
```