# squid-proxy

Esse proxy squid tem por objetivo criar um "proxy de proxy". O arquivo squid-proxy.yaml cria um proxy intermediário que encaminha as requisições para o proxy principal. Essa solução foi construída com a imagem ubuntu/squid:edge do squid.

## Deploy

Para realizar o teste local é necessário subir dois proxies, um que fará o papel do proxy principal (proxy.yaml) e outro que será o proxy intermediário (squid-proxy.yaml) que receberá as requisições antes de repassar para o proxy principal. Para o ambiente real/produtivo provavelmente só precisa subir o proxy intermediário(squid-proxy.yaml), pois possivelmente o proxy principal já existirá, sendo necessário apenas informar os dados de acesso do proxy principal no yaml squid-proxy.yaml.

### Proxy principal (apenas para testes locais)

Para realizar o deploy do proxy principal é necessário criar os recursos necessário para o proxy e o pod do proxy:

```
oc create -f proxy.yaml
```

Isso criará um ConfigMap onde ficará as configurações do squid, um PersistenceVolumeClaim para storage, um Service para acesso ao proxy e um Deployment com o Pod do proxy.

É necessário elevar a permissão do Deployment com o SCC anyuid:

```
oc create sa squid-sa
oc adm policy add-scc-to-user anyuid -z squid-sa
oc set sa deployment/proxy squid-sa
```

Após isso o Pod deve estar rodando corretamente.

Para que o proxy tenha um usuário de acesso basic para teste, copiar o arquivo passwords para o diretório /etc/squid/ dentro do Pod:

```
oc cp passwords pod-name:/etc/squid/passwords
```

Usuário admin com senha admin deve existir no proxy agora.

### Proxy intermediário

Agora é o momento de criar o proxy intermediário com squid, objetivo principal deste projeto. Antes de criar o squid-proxy, alterar a linha 21 do squid-proxy.yaml com as informações de acesso do proxy principal:

```
cache_peer PROXY_HOST parent PROXY_PORT 0 no-query default login=PROXY_USER:PROXY_PASSWORD
```
Obs.: Para o teste local o PROXY_HOST será o ip do service do proxy principal. Manter os demais valores.

Para o ambiente produtivo é aconselhável trocar a linha 13 de:

```
debug_options ALL,1 73,9
```
para:
```
debug_options ALL,1
```
Dessa forma a quantidade de linhas de log serão menores, pois não irá logar as requisições HTTP. Para mais informações do nível de log do squid, acessar: https://wiki.squid-cache.org/KnowledgeBase/DebugSections

Criar os recursos necessários para o squid-proxy:

```
oc create -f squid-proxy.yaml
```

Isso criará um ConfigMap onde ficará as configurações do squid, um PersistenceVolumeClaim para storage, um Service para acesso ao squid-proxy e um Deployment com o Pod do squid-proxy.

É necessário elevar a permissão do Deployment com o SCC anyuid:

```
oc create sa squid-sa
oc adm policy add-scc-to-user anyuid -z squid-sa
oc set sa deployment/squid-proxy squid-sa
```

Após isso o Pod deve estar rodando corretamente.

## Teste

Para realizar o teste se o squid-proxy está funcionando corretamente, abrir outro pod no namespace e executar:

```
curl http://www.google.com/ --proxy SQUID_PROXY_HOST:3128 | grep "<title>Google</title>"
```

O comando curl deve retornar o resultado.

Dentro do Pod do squid é possível visualizar os logs no arquivo var/log/squid/cache.log. O arquivo de configuração fica em etc/squid/squid.conf.