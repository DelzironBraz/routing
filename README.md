# Routing

## 1. Configurar os IPs nas máquinas:
Cada máquina vai precisar de um IP dentro da faixa 192.168.5.x ou 192.168.6.x. Supondo que você tem 3 máquinas com os IPs 192.168.5.1, 192.168.5.2, e 192.168.5.3 na interface eno1 e outras 3 máquinas com os IPs 192.168.6.1, 192.168.6.2, e 192.168.6.3 na interface eno2.

Em cada máquina (ou interface da rede), você pode atribuir os IPs manualmente:

Faça isso para cada máquina, mudando o último número do IP para atribuir 192.168.5.2, 192.168.5.3, 192.168.6.2, etc.

```shell
sudo ifconfig eno1 192.168.5.1 netmask 255.255.255.0 up
sudo ifconfig eno2 192.168.6.1 netmask 255.255.255.0 up
```

## 2. Adicionar rotas para a outra rede (192.168.6.0/24):
Em cada máquina da rede 192.168.5.0/24, você precisará adicionar uma rota para a rede 192.168.6.0/24. Para isso, será necessário configurar uma rota através de um gateway ou interface que esteja conectada à rede 192.168.6.0.

```shell
sudo route add -net 192.168.6.0 netmask 255.255.255.0 gw 192.168.5.1
```

Isso irá rotear pacotes da rede 192.168.5.0 para 192.168.6.0 usando 192.168.5.1 como gateway.

Repita o mesmo processo para a outra rede, adicionando rotas nas máquinas da rede 192.168.6.0/24 para se comunicar com a rede 192.168.5.0/24.

Exemplo para uma máquina na rede 192.168.6.0:

```shell
sudo route add -net 192.168.5.0 netmask 255.255.255.0 gw 192.168.6.1
```

## 3. Habilitar IP Forwarding (roteamento entre redes):
No roteador ou servidor que conecta as duas redes (o gateway), você precisará ativar o IP forwarding para que ele possa encaminhar pacotes entre as duas redes.

Execute o seguinte comando no gateway:

```shell
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
```

Isso permitirá que o sistema roteie pacotes entre as interfaces eno1 e eno2, possibilitando a comunicação entre máquinas nas redes 192.168.5.0/24 e 192.168.6.0/24.

## 4. Teste de conectividade:
Agora, você pode testar a conectividade com o comando ping entre as máquinas. Por exemplo, para testar se a máquina com IP 192.168.5.1 pode pingar a máquina com IP 192.168.6.1:

```shell
ping 192.168.6.1
```

Se o roteamento estiver configurado corretamente e o IP forwarding habilitado, as máquinas devem ser capazes de se comunicar entre as redes.

# Passos para configurar NAT usando iptables
## 1. Habilitar o encaminhamento de pacotes (IP forwarding):

Para que o servidor/roteador possa encaminhar pacotes entre as redes interna e externa, você precisa ativar o IP forwarding.

Execute o comando abaixo para habilitar o encaminhamento temporariamente:

```shell
sudo echo 1 > /proc/sys/net/ipv4/ip_forward
```

Para tornar isso permanente (sobreviver a reinicializações), adicione ou modifique a seguinte linha no arquivo /etc/sysctl.conf:

```shell
net.ipv4.ip_forward = 1
```

Aplique as alterações com o comando:

```shell
sudo sysctl -p
```

## 2. Configurar o NAT usando iptables:

Agora, você pode configurar o NAT para permitir que máquinas na rede interna (192.168.5.0/24) acessem a internet (ou outra rede pública) através da interface externa (eno2).

Execute o seguinte comando:

```shell
sudo iptables -t nat -A POSTROUTING -o eno2 -j MASQUERADE
```

Esse comando faz o seguinte:

- t nat: Especifica que você está trabalhando na tabela de NAT do iptables.
- A POSTROUTING: Adiciona uma regra na cadeia POSTROUTING, que altera os pacotes depois que eles foram roteados.
- o eno2: Aplica a regra à interface de saída eno2 (a interface conectada à internet ou rede pública).
- j MASQUERADE: Essa opção permite que o IP de saída seja mascarado, o que significa que o endereço IP privado das máquinas internas será substituído pelo endereço IP público da interface eno2.

## 3. Adicionar regras de firewall para permitir tráfego:

Para garantir que o tráfego das máquinas internas possa sair para a internet e que as respostas voltem corretamente, adicione as seguintes regras:

Permitir tráfego de entrada relacionado ou estabelecido:

```shell
sudo iptables -A FORWARD -i eno2 -o eno1 -m state --state RELATED,ESTABLISHED -j ACCEPT
```

Permitir tráfego de saída da rede interna para a externa:

```shell
sudo iptables -A FORWARD -i eno1 -o eno2 -j ACCEPT
```

## 4. Salvar as regras do iptables (opcional):

As regras do iptables podem ser temporárias e perdidas após a reinicialização. Para salvá-las permanentemente, use o comando abaixo (dependendo da distribuição Linux que você usa):

No Ubuntu/Debian, instale o pacote iptables-persistent:

```shell
sudo apt install iptables-persistent
sudo service netfilter-persistent save
```

# Testar o NAT
Após configurar o NAT, você pode testar o serviço nas máquinas da rede interna (192.168.5.0/24):

## 1. Definir o gateway nas máquinas da rede interna:
Certifique-se de que as máquinas na rede 192.168.5.0/24 estejam configuradas para usar o IP da interface interna do servidor NAT (por exemplo, 192.168.5.1) como o gateway.

## 2. Testar a conectividade com a rede externa (ping ou curl):
A partir de uma máquina da rede interna (por exemplo, 192.168.5.2), tente acessar a internet ou pingar um IP externo:

```shell
ping 8.8.8.8
```