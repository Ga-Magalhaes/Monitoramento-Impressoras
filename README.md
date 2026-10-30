# Monitoramento de Impressoras

Repositório privado com as configurações do monitoramento de impressoras via SNMP, Prometheus, snmp_exporter e Grafana.

## Servidor

* Sistema: FreeBSD
* Hostname: srvd22FREE
* IP do servidor: 192.168.0.22
* Prometheus: http://127.0.0.1:9090
* snmp_exporter: http://127.0.0.1:9116
* Grafana: http://192.168.0.22:3000

## Arquivos principais

| Arquivo                | Destino no servidor                            | Função                                 |
| ---------------------- | ---------------------------------------------- | -------------------------------------- |
| configs/prometheus.yml | /usr/local/etc/prometheus.yml                  | Configuração principal do Prometheus   |
| configs/snmp.yml       | /usr/local/etc/snmp_exporter/snmp.yml          | Módulos SNMP usados pelo snmp_exporter |
| configs/printers.yml   | /usr/local/etc/prometheus/file_sd/printers.yml | Inventário das impressoras             |

## Serviços

Validar configuração do Prometheus:

```sh
promtool check config /usr/local/etc/prometheus.yml
```

Reiniciar Prometheus:

```sh
sudo service prometheus restart
```

Reiniciar snmp_exporter:

```sh
sudo service snmp_exporter restart
```

Reiniciar Grafana:

```sh
sudo service grafana restart
```

## Como cadastrar uma nova impressora

### 1. Confirmar IP da impressora

Testar ping:

```sh
ping -c 4 IP_DA_IMPRESSORA
```

Exemplo:

```sh
ping -c 4 192.168.0.213
```

### 2. Testar SNMP

Testar se a impressora responde SNMP:

```sh
snmpget -v2c -c empresa -On IP_DA_IMPRESSORA 1.3.6.1.2.1.1.1.0
```

Testar contador total:

```sh
snmpget -v2c -c empresa -On IP_DA_IMPRESSORA 1.3.6.1.2.1.43.10.2.1.4.1.1
```

Se der timeout, verificar na interface web da impressora:

* SNMP v1/v2c habilitado
* GET Community Name: empresa
* IP Filtering / Allowed Managers liberando o servidor 192.168.0.22
* Porta UDP 161 liberada

### 3. Adicionar no inventário

Editar:

```sh
sudo ee /usr/local/etc/prometheus/file_sd/printers.yml
```

Modelo:

```yaml
- targets:
    - IP_DA_IMPRESSORA
  labels:
    device_name: "NOME_DA_IMPRESSORA"
    vendor: "Xerox"
    location: "LOCAL"
    role: "principal"
    snmp_module: "printer_mib"
    snmp_auth: "empresa_v2"
```

Exemplo:

```yaml
- targets:
    - 192.168.0.213
  labels:
    device_name: "XEROX_3655_CL"
    vendor: "Xerox"
    location: "CL"
    role: "principal"
    snmp_module: "printer_mib"
    snmp_auth: "empresa_v2"
```

### 4. Se for Xerox colorida com contador de impressão em cores

Adicionar também no job `xerox_billing` dentro do `/usr/local/etc/prometheus.yml`.

Modelo:

```yaml
      - targets:
          - IP_DA_IMPRESSORA
        labels:
          device_name: "NOME_DA_IMPRESSORA"
          vendor: "Xerox"
          location: "LOCAL"
```

Esse job coleta:

* Total de impressões
* Impressões em preto e branco
* Impressões em cores

### 5. Validar e reiniciar

```sh
promtool check config /usr/local/etc/prometheus.yml
sudo service prometheus restart
```

### 6. Testar no Prometheus

Verificar se a impressora está cadastrada:

```sh
curl -s -G 'http://127.0.0.1:9090/api/v1/query' \
  --data-urlencode 'query=up{job="printers_snmp"}'
```

Verificar contador total:

```sh
curl -s -G 'http://127.0.0.1:9090/api/v1/query' \
  --data-urlencode 'query=prtMarkerLifeCount{job="printers_snmp",instance="IP_DA_IMPRESSORA"}'
```

Verificar níveis de toner:

```sh
curl -s -G 'http://127.0.0.1:9090/api/v1/query' \
  --data-urlencode 'query=prtMarkerSuppliesLevel{job="printers_snmp",instance="IP_DA_IMPRESSORA"}'
```

Verificar impressão em cores, para Xerox colorida:

```sh
curl -s -G 'http://127.0.0.1:9090/api/v1/query' \
  --data-urlencode 'query=xerox_billing_color_impressions{job="xerox_billing",instance="IP_DA_IMPRESSORA"}'
```

## OIDs importantes

### Printer-MIB padrão

Contador total:

```text
1.3.6.1.2.1.43.10.2.1.4.1.1
```

Descrição dos suprimentos:

```text
1.3.6.1.2.1.43.11.1.1.6
```

Capacidade máxima do suprimento:

```text
1.3.6.1.2.1.43.11.1.1.8
```

Nível atual do suprimento:

```text
1.3.6.1.2.1.43.11.1.1.9
```

### Xerox Billing

Total de impressões:

```text
1.3.6.1.4.1.253.8.53.13.2.1.6.1.20.1
```

Impressões em preto e branco:

```text
1.3.6.1.4.1.253.8.53.13.2.1.6.1.20.34
```

Impressões em cores:

```text
1.3.6.1.4.1.253.8.53.13.2.1.6.1.20.33
```

## Problemas comuns

### A impressora responde ping, mas não aparece no Grafana

Testar SNMP:

```sh
snmpget -v2c -c empresa -On IP_DA_IMPRESSORA 1.3.6.1.2.1.1.1.0
```

Se der timeout, verificar:

* SNMP habilitado
* Community correta
* IP Filtering
* Liberação do IP do servidor 192.168.0.22

### Aparece no dashboard, mas mostra No data

Isso normalmente significa que a impressora está cadastrada no Prometheus, mas não está entregando métricas SNMP naquele momento.

### Impressora offline aparece no dashboard

Isso é esperado. O dashboard usa `up{job="printers_snmp"}` para listar também impressoras cadastradas que estão desligadas ou sem SNMP.

### Xerox 3655 sem código PN no SNMP

Algumas Xerox 3655 mostram o toner como:

```text
Toner Cartridge;SN...
```

sem PN. Para essas impressoras, o código usado manualmente é:

```text
106R02742
```

## Backup das configurações

Para salvar as configurações atuais no repositório:

```sh
cd ~/repos/monitoramento-impressoras

sudo cp /usr/local/etc/prometheus.yml configs/prometheus.yml
sudo cp /usr/local/etc/snmp_exporter/snmp.yml configs/snmp.yml
sudo cp /usr/local/etc/prometheus/file_sd/printers.yml configs/printers.yml

sudo chown -R gabrielm:wheel .
```

Depois versionar:

```sh
git status
git add .
git commit -m "Atualiza configuracoes do monitoramento de impressoras"
git push
```

## Restauração

Para restaurar as configurações a partir do repositório:

```sh
cd ~/repos/monitoramento-impressoras

sudo cp configs/prometheus.yml /usr/local/etc/prometheus.yml
sudo cp configs/snmp.yml /usr/local/etc/snmp_exporter/snmp.yml
sudo cp configs/printers.yml /usr/local/etc/prometheus/file_sd/printers.yml

promtool check config /usr/local/etc/prometheus.yml
sudo service snmp_exporter restart
sudo service prometheus restart
```
