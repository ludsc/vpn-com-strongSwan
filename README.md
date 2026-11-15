# Laboratório: VPN IPsec com StrongSwan e Docker 🛡️

Este repositório contém um laboratório prático para configurar uma comunicação segura entre dois hosts virtuais (Cliente e Servidor) utilizando o protocolo **IPsec** e negociação **IKEv2** através da ferramenta **StrongSwan**. O ambiente é totalmente orquestrado via Docker.

## 🎯 Objetivo
Demonstrar a criação de um túnel VPN seguro (cifrado e autenticado) implementando uma Infraestrutura de Chaves Públicas (PKI) para autenticação mútua via certificados digitais.

## 🏗️ Arquitetura e Pré-requisitos
- **Docker** e **Docker Compose** instalados.
- Dois contêineres baseados em Alpine Linux (`vpn-server` e `vpn-client`).
- A capacidade `NET_ADMIN` e `SYS_MODULE` ativadas no Docker, essenciais para que os contêineres possam manipular as pilhas de rede e módulos de criptografia do Kernel (ESP/AH).

---

## 🚀 Passo a Passo para Reprodução

### Fase 1: Preparação do Ambiente

1. Crie a estrutura de diretórios para mapeamento dos volumes:
   ```bash
   mkdir -p server-conf/{cacerts,certs,private}
   mkdir -p client-conf/{cacerts,certs,private}
   ```

2. Crie o arquivo `docker-compose.yml` na raiz do projeto:
   ```yaml
   version: '3.8'

   services:
     vpn-server:
       image: alpine:latest
       container_name: strongswan-server
       cap_add:
         - NET_ADMIN
         - SYS_MODULE
       volumes:
         - ./server-conf:/etc/ipsec.d
       command: sh -c "apk update && apk add strongswan iptables && tail -f /dev/null"
       networks:
         lab_net:
           ipv4_address: 10.10.10.10

     vpn-client:
       image: alpine:latest
       container_name: strongswan-client
       cap_add:
         - NET_ADMIN
         - SYS_MODULE
       volumes:
         - ./client-conf:/etc/ipsec.d
       command: sh -c "apk update && apk add strongswan iptables && tail -f /dev/null"
       networks:
         lab_net:
           ipv4_address: 10.10.10.20

   networks:
     lab_net:
       driver: bridge
       ipam:
         config:
           - subnet: 10.10.10.0/24
   ```

3. Suba o ambiente:
   ```bash
   docker-compose up -d
   docker exec strongswan-server sh -c "ln -sf /etc/ipsec.d/ipsec.conf /etc/ipsec.conf && ln -sf /etc/ipsec.d/ipsec.secrets /etc/ipsec.secrets && ipsec restart"
   docker exec strongswan-client sh -c "ln -sf /etc/ipsec.d/ipsec.conf /etc/ipsec.conf && ln -sf /etc/ipsec.d/ipsec.secrets /etc/ipsec.secrets && ipsec restart"

   #validar estabelecimento do tunel, conexao e criptografia
   docker exec strongswan-server ipsec status

   docker exec strongswan-client ping -c 4 10.10.10.10

   docker exec strongswan-server ip xfrm state
   
   ```

### Fase 2: Infraestrutura de Chaves Públicas (PKI)

Utilizaremos o utilitário `pki` do StrongSwan para gerar os certificados.

**1. Gerar a Autoridade Certificadora (Root CA):**
```bash
docker exec strongswan-server pki --gen --type rsa --size 4096 --outform pem > server-conf/private/ca-key.pem
docker exec strongswan-server pki --self --ca --lifetime 3650 --in /etc/ipsec.d/private/ca-key.pem --type rsa --dn "CN=Lab VPN CA" --outform pem > server-conf/cacerts/ca-cert.pem
```

**2. Gerar o Certificado do Servidor:**
```bash
docker exec strongswan-server pki --gen --type rsa --size 4096 --outform pem > server-conf/private/server-key.pem
docker exec strongswan-server pki --pub --in /etc/ipsec.d/private/server-key.pem --type rsa | docker exec -i strongswan-server pki --issue --lifetime 1825 --cacert /etc/ipsec.d/cacerts/ca-cert.pem --cakey /etc/ipsec.d/private/ca-key.pem --dn "CN=10.10.10.10" --san 10.10.10.10 --flag serverAuth --flag ikeIntermediate --outform pem > server-conf/certs/server-cert.pem
```

**3. Gerar o Certificado do Cliente:**
```bash
docker exec strongswan-server pki --gen --type rsa --size 4096 --outform pem > server-conf/private/client-key.pem
docker exec strongswan-server pki --pub --in /etc/ipsec.d/private/client-key.pem --type rsa | docker exec -i strongswan-server pki --issue --lifetime 1825 --cacert /etc/ipsec.d/cacerts/ca-cert.pem --cakey /etc/ipsec.d/private/ca-key.pem --dn "CN=vpn-client" --san vpn-client --outform pem > server-conf/certs/client-cert.pem
```

**4. Distribuir as chaves do cliente e ajustar permissões (Importante!):**
```bash
cp server-conf/cacerts/ca-cert.pem client-conf/cacerts/
cp server-conf/certs/client-cert.pem client-conf/certs/
mv server-conf/private/client-key.pem client-conf/private/

# O IPsec recusa chaves privadas com leitura pública
sudo chmod 600 server-conf/private/server-key.pem
sudo chmod 600 client-conf/private/client-key.pem
```

### Fase 3: Configuração do IPsec (IKEv2)

**1. Servidor (`server-conf/ipsec.conf`):**
*Nota: No jargão do StrongSwan, `left` refere-se à máquina local e `right` à máquina remota.*
```text
config setup
    charondebug="ike 1, knl 1, cfg 0"
    uniqueids=no

conn ikev2-vpn
    auto=add
    compress=no
    type=tunnel
    keyexchange=ikev2
    fragmentation=yes
    forceencaps=yes
    dpdaction=clear
    dpddelay=300s
    rekey=no
    left=%any
    leftid="CN=10.10.10.10"
    leftcert=server-cert.pem
    leftsendcert=always
    leftsubnet=0.0.0.0/0
    right=%any
    rightid="CN=vpn-client"
    rightauth=pubkey
    rightsourceip=10.10.10.100
    rightdns=8.8.8.8
```

**2. Segredos do Servidor (`server-conf/ipsec.secrets`):**
```text
: RSA "server-key.pem"
```

**3. Cliente (`client-conf/ipsec.conf`):**
```text
config setup
    charondebug="ike 1, knl 1, cfg 0"

conn ikev2-vpn
    auto=start
    type=tunnel
    keyexchange=ikev2
    left=%any
    leftid="CN=vpn-client"
    leftcert=client-cert.pem
    leftsourceip=%config
    right=10.10.10.10
    rightid="CN=10.10.10.10"
    rightsubnet=0.0.0.0/0
    rightauth=pubkey
```

**4. Segredos do Cliente (`client-conf/ipsec.secrets`):**
```text
: RSA "client-key.pem"
```

### Fase 4: Ativando e Validando a VPN

Aplique as configurações em ambos os contêineres:
```bash
docker exec strongswan-server sh -c "ln -sf /etc/ipsec.d/ipsec.conf /etc/ipsec.conf && ln -sf /etc/ipsec.d/ipsec.secrets /etc/ipsec.secrets && ipsec restart"

docker exec strongswan-client sh -c "ln -sf /etc/ipsec.d/ipsec.conf /etc/ipsec.conf && ln -sf /etc/ipsec.d/ipsec.secrets /etc/ipsec.secrets && ipsec restart"
```

**Testes:**
1. Verifique se o túnel foi estabelecido:
   ```bash
   docker exec strongswan-server ipsec status
   ```
   *(A saída deve exibir `ESTABLISHED` e indicar a instalação do `TUNNEL`)*.

2. Teste a conectividade por dentro do túnel:
   ```bash
   docker exec strongswan-client ping -c 4 10.10.10.10
   ```

3. Verifique se o Kernel está aplicando a criptografia (ESP):
   ```bash
   docker exec strongswan-server ip xfrm state
   ```
   *(Isto listará as chaves criptográficas ativas no modo de encapsulamento).*
