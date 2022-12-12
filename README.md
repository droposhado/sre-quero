# sre-quero

Esta é uma solução que visa a simplicidade de coleta com posterior processamento dos logs de acesso da aplicação tida como principal, utilizando para isso o promtail, que é um cliente do Grafana Loki para coleta de logs no Kubernetes, com uso mais comum como **DaemonSet**, porém é possível fazer o uso em modo standalone (como sidecar).

Os logs coletados podem ser visualizados no Grafana e quando o container contendo o promtail tem acesso externo configurado é possível obter métricas sob o path `/metrics`.

A aplicação enviada junto na pasta de exemplo é a [droposhado/sre-quero-application](https://github.com/droposhado/sre-quero-application), feita para gerar logs, com configuração adicional para a pasta de log da aplicação ser um volume e o log gerado pelo unicorn com o último campo sendo o tempo da requisação processada pelo próprio gunicorn.

## Coleta de logs

Como é sidecar é preciso que a aplicação escreva em disco os logs de acesso, como na pasta de exemplo, se configura uma pasta onde ele ser`/var/log/app/`. 

## Compartilhamento de logs entre containers dentro do pod

Esta configuração é a que se espera no `deployment.yml` e esta pasta de log é colocada como um volume no `pod`, então é reciso declarar um volume em `spec.volumes`:

```yml
spec:
  [...]
  volumes:
    - name: shared-logs
      emptyDir: {}
```

Agora deve ser utilizado em cada um dos containers, sendo declarado em `spec.containers.[n].volumeMounts`:

```yml
spec:
  containers:
    - name: main-application
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: sidecar-container
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
```

Resultado no exemplo abaixo:

```yml
metadata:
  name: simple-webapp
  labels:
    app: webapp
spec:
  containers:
    - name: main-application
      image: quay.io/droposhado/application:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: sidecar-container
      image: grafana/promtail:2.0.0
      args:
        - -config.file=/etc/promtail/promtail.yaml
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
  volumes:
    - name: shared-logs
      emptyDir: {}
``` 

## Configuração do promtail

A configuração do promtail indicada no arquivo `deployment.yml` é armazenada numa variável, demonstrada em `spec.containers.[n].args`

```yml
spec:
  containers:
    - args:
        - -config.file=/etc/promtail/promtail.yaml
```

O valor dessa variável colocado no arquivo `configmap.yaml`, é dado em `data.'promtail.yaml'`, todo valor é provido pela interface do Grafana, na configuração do Loki, e contempla:

- `server.[n]`: ficam configurações relativas ao servidor;
  - `server.http_listen_port`: a porta para exposição de métricas;
- `client.[n]`: configurações de onde e como enviar os logs, que podem ser multiplos recipientes;
  - `client.url`: indica a url de push do Loki;
- `positions`: arquivo temporário que armazena a última posição de envio;
- `scrape_configs`: define as fontes onde devem ser buscados os logs;
- `scrape_configs.[n].job_name`: nome do log apresentado na interface;
- `scrape_configs.[n].static_configs`: dentre os tipos disponíveis, este indica um arquivo no disco;
- `scrape_configs.[n].targets.[n]`: indica as máquinas onde devem ser lido(s) o(s) arquivo(s);
- `scrape_configs.[n].labels.[n]`: onde se indicam qual ou quais arquivos vão ser lidos;
- `scrape_configs.[n].labels.[n]__path__`: indica qual o arquivo.

## Exposição de portas do promtail

Na parte do container é preciso expor a porta do serviço (`deployment.yml`) pra acesso externo `spec.containers.[n].ports`

```yml 
spec:
  containers:
    - name: sidecar-container
      ports:
        - containerPort: 9080
          name: promtail-http
```

E adicionar uma entrada no `service.yml`:

```yml
spec:
  ports:
    - name: promtail-http
      protocol: TCP
      port: 9080
      targetPort: promtail-http
```

Sendo que `spec.containers.[n].ports.name` deve ser igual ao `spec.ports.[n].targetPort`, onde `spec.ports.[n].port` deve ser o mesmo que declarado no `configmap.yml` no `promtail.yml` em `server.http_listen_port`.

Aplicação está configurada com o container de sidecar

## Referências

- [TUDO O QUE VOCÊ PRECISA SABER SOBRE KUBERNETES PARA PASSAR NA ENTREVISTA!](https://www.youtube.com/watch?v=zEOeukcJl6E)
- [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [Grafana Loki Fundamentals](https://grafana.com/docs/loki/latest/fundamentals/overview/)
- [Promtail](https://grafana.com/docs/loki/latest/clients/promtail/)
- [Install Promtail](https://grafana.com/docs/loki/latest/clients/promtail/installation/)
- [gunicorn - docs - Settings](https://docs.gunicorn.org/en/latest/settings.html#access-log-format)
- [Kubernetes sidecar container examples](https://www.golinuxcloud.com/kubernetes-sidecar-container/)
- [Kubernetes Sidecar Container | Best Practices and Examples](https://www.containiq.com/post/kubernetes-sidecar-container)
- [Using sidecar containers in k8s](https://sreeraj.dev/sidecar-containers/)
- [Deploy Promtail as a Sidecar to you Main App](https://techdaily.info/deploy-promtail-as-a-sidecar-to-you-main-app)
