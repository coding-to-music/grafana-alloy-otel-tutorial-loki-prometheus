# grafana-alloy-otel-tutorial-loki-prometheus

# 🚀 Configure Alloy to collect logs from your local machine, filter non-essential log lines, send them to Loki, and use Grafana to explore the results. 🚀

https://github.com/coding-to-music/grafana-alloy-otel-tutorial-loki-prometheus

From / By https://grafana.com/docs/alloy/latest/tutorials/send-logs-to-loki

## Environment variables:

```java

```

## GitHub

```java
git init
git add .
git remote remove origin
git commit -m "first commit"
git branch -M main
git remote add origin git@github.com:coding-to-music/grafana-alloy-otel-tutorial-loki-prometheus.git
git push -u origin main
```

## Grafana Alloy home directory

```java
cd /etc/alloy

sudo vi config.alloy
```

# Store secrets so Grafana Alloy can reference them every time it is reloaded and not needed to be hard-coded

To ensure your Grafana secrets are auto-loaded on reboot or restart of Alloy, you can store them in a file that is sourced during the boot process. Here’s a common approach to achieve this:

Add any secrets to the alloy userid startup defaults file `/etc/default/alloy`

```java
HOSTNAME=
GRAFANA_LOKI_URL=
GRAFANA_LOKI_USERNAME=
GRAFANA_LOKI_PASSWORD=
GRAFANA_PROM_URL=
GRAFANA_PROM_USERNAME=
GRAFANA_PROM_PASSWORD=
```

paste them into the `/etc/default/alloy` file:

```java
sudo vi /etc/default/alloy
```

set the permissions restrictively

```java
sudo chmod 600 /etc/default/alloy
sudo chown alloy:alloy /etc/default/alloy
```

# restart alloy so it will see the environment values

sudo systemctl restart alloy

# view the status

sudo systemctl status alloy

# if you just want to reload config.alloy then call this:

curl -X POST http://localhost:12345/-/reload

# if any errors take a look at the log

journalctl -u alloy.service -n 50

# or

journalctl -u alloy -n 50

````

Use the environment values like this:

```java
loki.relabel "logs_integrations" {
  forward_to = [loki.write.grafana_cloud_loki.receiver]

  rule {
    replacement = sys.env("ALLOY_HOSTNAME")
    target_label = "instance"
  }

  rule {
    target_label = "job"
    replacement = "integrations/alloy"
  }
}

prometheus.remote_write "metrics_service" {
  endpoint {
    url = sys.env("GRAFANA_PROM_URL")

    basic_auth {
      username = sys.env("GRAFANA_PROM_USERNAME")
      password = sys.env("GRAFANA_PROM_PASSWORD")
    }
  }
}

loki.write "grafana_cloud_loki" {
  endpoint {
    url = sys.env("GRAFANA_LOKI_URL")

    basic_auth {
      username = sys.env("GRAFANA_LOKI_USERNAME")
      password = sys.env("GRAFANA_LOKI_PASSWORD")
    }
  }
}
````

If needed Reload System Daemons:

```java
sudo systemctl daemon-reload
```

Restart Your Service:

```java
sudo systemctl restart alloy.service
```

By following these steps, your Grafana secrets should be loaded automatically on system reboot or when the Alloy service is restarted.

## How to use Salt Stack to distribute the config.alloy

To create a Salt Stack job that distributes an updated `config.alloy` file, you can use Salt's file management capabilities. Here's a basic approach to accomplish this:

First, create a Salt State file (e.g., `alloy_config.sls`) with the following content:

```java
/etc/alloy/config.alloy:
  file.managed:
    - source: salt://alloy/config.alloy
    - user: root
    - group: root
    - mode: 644
    - makedirs: True

reload_alloy_config:
  cmd.run:
    - name: curl -X POST http://localhost:12345/-/reload
    - onchanges:
      - file: /etc/alloy/config.alloy
```

Place your updated `config.alloy` file in the Salt file server, typically in `/srv/salt/alloy/config.alloy` on the Salt master.

Apply the state to your minions using the Salt command:

```java
salt '*' state.apply alloy_config
```

This Salt State does the following:

- It manages the `/etc/alloy/config.alloy` file on the minions, ensuring it matches the source file on the Salt master.
- It sets appropriate permissions on the file.
- It runs the `curl` command to reload the Alloy configuration, but only if the file has changed.
- Remember to adjust the file paths and permissions as necessary for your specific setup.

For more advanced usage, you might want to consider using Salt's template engine (Jinja) to dynamically generate the config file based on pillar data or other variables.

For more information on Salt Stack's file management and state system, you can refer to the Salt Stack file management documentation. https://docs.saltproject.io/en/latest/ref/states/all/salt.states.file.html

Note that this approach assumes that the Alloy service is running and accessible on `localhost:12345` on all minions. If this is not the case, you may need to adjust the reload command or add additional checks.

## Use Grafana Alloy to send logs to Loki

https://grafana.com/docs/alloy/latest/tutorials/send-logs-to-loki

This tutorial shows you how to configure Alloy to collect logs from your local machine, filter non-essential log lines, send them to Loki, and use Grafana to explore the results.

### Before you begin

To complete this tutorial:

- You must have a basic understanding of Alloy and telemetry collection in general.
- You should be familiar with Prometheus, PromQL, Loki, LogQL, and basic Grafana navigation.
  Tip

Alternatively, you can try out this example in the interactive learning environment: Sending Logs to Loki.

It’s a fully configured environment with all the dependencies already installed.

Interactive
![image](/images/Alloy-Interactive-Learning-Environment-Doc-Banner.png)

### Install Alloy and start the service

This tutorial requires a Linux or macOS environment with Docker installed.

### Linux

Install and run Alloy on Linux.

- Install Alloy. https://grafana.com/docs/alloy/latest/set-up/install/linux/
- Run Alloy. https://grafana.com/docs/alloy/latest/set-up/run/linux/

### macOS

Install and run Alloy on macOS.

- Install Alloy. https://grafana.com/docs/alloy/latest/set-up/install/macos/
- Run Alloy. https://grafana.com/docs/alloy/latest/set-up/run/macos/

You can access the Alloy UI at http://localhost:12345.

### Set up a local Grafana instance

In this tutorial, you configure Alloy to collect logs from your local machine and send them to Loki. You can use the following Docker Compose file to set up a local Grafana instance. This Docker Compose file includes Loki and Prometheus configured as data sources.

Create a directory and save the Docker Compose file as `docker-compose.yml`.

```java
mkdir alloy-tutorial
cd alloy-tutorial
touch docker-compose.yml
```

Copy the following Docker Compose file into `docker-compose.yml`.

```java
 version: '3'
 services:
   loki:
     image: grafana/loki:3.0.0
     ports:
       - "3100:3100"
     command: -config.file=/etc/loki/local-config.yaml
   prometheus:
     image: prom/prometheus:v2.47.0
     command:
       - --web.enable-remote-write-receiver
       - --config.file=/etc/prometheus/prometheus.yml
     ports:
       - "9090:9090"
   grafana:
     environment:
       - GF_PATHS_PROVISIONING=/etc/grafana/provisioning
       - GF_AUTH_ANONYMOUS_ENABLED=true
       - GF_AUTH_ANONYMOUS_ORG_ROLE=Admin
     entrypoint:
       - sh
       - -euc
       - |
         mkdir -p /etc/grafana/provisioning/datasources
         cat <<EOF > /etc/grafana/provisioning/datasources/ds.yaml
         apiVersion: 1
         datasources:
         - name: Loki
           type: loki
           access: proxy
           orgId: 1
           url: http://loki:3100
           basicAuth: false
           isDefault: false
           version: 1
           editable: false
         - name: Prometheus
           type: prometheus
           orgId: 1
           url: http://prometheus:9090
           basicAuth: false
           isDefault: true
           version: 1
           editable: false
         EOF
         /run.sh
     image: grafana/grafana:11.0.0
     ports:
       - "3000:3000"
```

To start the local Grafana instance, run the following command.

```java
docker compose up -d
```

Note

If you encounter the following error when you start your Docker container, docker: 'compose' is not a docker command, use the command `docker-compose up` to start your Docker container.

Open http://localhost:3000 in your browser to access the Grafana UI.

### Configure Alloy

After the local Grafana instance is set up, the next step is to configure Alloy. You use components in the `config.alloy` file to tell Alloy which logs you want to scrape, how you want to process that data, and where you want the data sent.

The examples run on a single host so that you can run them on your laptop or in a Virtual Machine. You can try the examples using a `config.alloy` file and experiment with the examples.

### Create a `config.alloy` file

Create a `config.alloy` file within your current working directory.

```java
touch config.alloy
```

### First component: Log files

Copy and paste the following component configuration at the top of the file.

```java
 local.file_match "local_files" {
     path_targets = [{"__path__" = "/var/log/*.log"}]
     sync_period = "5s"
 }
```

This configuration creates a local.file_match component named local_files which does the following:

- It tells Alloy which files to source.
- It checks for new files every 5 seconds.

### Second component: Scraping

Copy and paste the following component configuration below the previous component in your `config.alloy` file:

```java
  loki.source.file "log_scrape" {
    targets    = local.file_match.local_files.targets
    forward_to = [loki.process.filter_logs.receiver]
    tail_from_end = true
  }
```

This configuration creates a loki.source.file component named log_scrape which does the following:

- It connects to the local_files component as its source or target.
- It forwards the logs it scrapes to the receiver of another component called filter_logs.
- It provides extra attributes and options to tail the log files from the end so you don’t ingest the entire log file history.

### Third component: Filter non-essential logs

Filtering non-essential logs before sending them to a data source can help you manage log volumes to reduce costs.

The following example demonstrates how you can filter out or drop logs before sending them to Loki.

Copy and paste the following component configuration below the previous component in your `config.alloy` file:

```java
  loki.process "filter_logs" {
    stage.drop {
        source = ""
        expression  = ".*Connection closed by authenticating user root"
        drop_counter_reason = "noisy"
      }
    forward_to = [loki.write.grafana_loki.receiver]
    }
```

The loki.process component allows you to transform, filter, parse, and enrich log data. Within this component, you can define one or more processing stages to specify how you would like to process log entries before they’re stored or forwarded.

This configuration creates a loki.process component named filter_logs which does the following:

- It receives scraped log entries from the default log_scrape component.
- It uses the stage.drop block to define what to drop from the scraped logs.
- It uses the expression parameter to identify the specific log entries to drop.
- It uses an optional string label drop_counter_reason to show the reason for dropping the log entries.
- It forwards the processed logs to the receiver of another component called grafana_loki.

The `loki.process` documentation provides more comprehensive information on processing logs.

### Fourth component: Write logs to Loki

Copy and paste this component configuration below the previous component in your config.alloy file.

```java
  loki.write "grafana_loki" {
    endpoint {
      url = "http://localhost:3100/loki/api/v1/push"

      // basic_auth {
      //  username = "admin"
      //  password = "admin"
      // }
    }
  }
```

This final component creates a `loki.write` component named grafana_loki that points to http://localhost:3100/loki/api/v1/push.

This completes the simple configuration pipeline.

Tip

The `basic_auth` block is commented out because the local `docker compose` stack doesn’t require it. It’s included in this example to show how you can configure authorization for other environments. For further authorization options, refer to the `loki.write` component reference.

With this configuration, Alloy connects directly to the Loki instance running in the Docker container.

### Reload the configuration

Copy your local `config.alloy` file into the default Alloy configuration file location.

```java
sudo cp config.alloy /etc/alloy/config.alloy
```

Call the `/-/reload` endpoint to tell Alloy to reload the configuration file without a system service restart.

```java
curl -X POST http://localhost:12345/-/reload
```

Tip

This step uses the Alloy UI on `localhost` port `12345`. If you chose to run Alloy in a Docker container, make sure you use the `--server.http.listen-addr=0.0.0.0:12345` argument. If you don’t use this argument, the debugging UI won’t be available outside of the Docker container. https://grafana.com/docs/alloy/latest/troubleshoot/debug/#alloy-ui

Optional: You can do a system service restart Alloy and load the configuration file.

```java
sudo systemctl reload alloy
```

Inspect your configuration in the Alloy UI

Open http://localhost:12345 and click the Graph tab at the top. The graph should look similar to the following:

Your configuration in the Alloy UI
![image](/images/Inspect-your-config-in-the-Alloy-UI-image.png)

The Alloy UI shows you a visual representation of the pipeline you built with your Alloy component configuration.

You can see that the components are healthy, and you are ready to explore the logs in Grafana.

### Log in to Grafana and explore Loki logs

Open http://localhost:3000/explore to access Explore feature in Grafana.

Select Loki as the data source and click the Label Browser button to select a file that Alloy has sent to Loki.

Here you can see that logs are flowing through to Loki as expected, and the end-to-end configuration was successful.

Logs reported by Alloy in Grafana
![image](/images/loki-logs.png)

### Summary

You have installed and configured Alloy, and sent logs from your local host to your local Grafana stack.

In the next tutorial, you learn more about configuration concepts and metrics.

# Use Grafana Alloy to send metrics to Prometheus

https://grafana.com/docs/alloy/latest/tutorials/send-metrics-to-prometheus/

In the previous tutorial, you learned how to configure Alloy to collect and process logs from your local machine and send them to Loki.

This tutorial shows you how to configure Alloy to collect and process metrics from your local machine, send them to Prometheus, and use Grafana to explore the results.

## Before you begin

To complete this tutorial:

- You must have a basic understanding of Alloy and telemetry collection in general.
- You should be familiar with Prometheus, PromQL, Loki, LogQL, and basic Grafana navigation.
- You must complete the previous tutorial to prepare the following prerequisites:
  - Install Alloy and start the service in your environment.
  - Set up a local Grafana instance.
  - Create a `config.alloy` file.

Tip

Alternatively, you can try out this example in the interactive learning environment: Sending metrics to Prometheus.

It’s a fully configured environment with all the dependencies already installed.

Interactive
![image](/images/Alloy-Interactive-Learning-Environment-Doc-Banner2.png)

## Configure Alloy

In this tutorial, you configure Alloy to collect metrics and send them to Prometheus.

You add components to your `config.alloy` file to tell Alloy which metrics you want to scrape, how you want to process that data, and where you want the data sent.

The following steps build on the `config.alloy` file you created in the previous tutorial.

## First component: Scraping

Paste the following component configuration at the top of your `config.alloy` file:

```java
prometheus.exporter.unix "local_system" { }

prometheus.scrape "scrape_metrics" {
  targets         = prometheus.exporter.unix.local_system.targets
  forward_to      = [prometheus.relabel.filter_metrics.receiver]
  scrape_interval = "10s"
}
```

This configuration creates a prometheus.scrape component named scrape_metrics which does the following:

- It connects to the `local_system` component as its source or target.
- It forwards the metrics it scrapes to the receiver of another component called `filter_metrics`.
- It tells Alloy to scrape metrics every 10 seconds.

## Second component: Filter metrics

Filtering non-essential metrics before sending them to a data source can help you reduce costs and allow you to focus on the data that matters most.

The following example demonstrates how you can filter out or drop metrics before sending them to Prometheus.

Paste the following component configuration below the previous component in your `config.alloy` file:

```java
prometheus.relabel "filter_metrics" {
  rule {
    action        = "drop"
    source_labels = ["env"]
    regex         = "dev"
  }

  forward_to = [prometheus.remote_write.metrics_service.receiver]
}
```

The `prometheus.relabel` component is commonly used to filter Prometheus metrics or standardize the label set passed to one or more downstream receivers. You can use this component to rewrite the label set of each metric sent to the receiver. Within this component, you can define rule blocks to specify how you would like to process metrics before they’re stored or forwarded.

This configuration creates a `prometheus.relabel` component named filter_metrics which does the following:

- It receives scraped metrics from the scrape_metrics component.
- It tells Alloy to drop metrics that have an "env" label equal to "dev".
- It forwards the processed metrics to the receiver of another component called metrics_service.

## Third component: Write metrics to Prometheus

Paste the following component configuration below the previous component in your `config.alloy` file:

```java
prometheus.remote_write "metrics_service" {
    endpoint {
        url = "http://localhost:9090/api/v1/write"

        // basic_auth {
        //   username = "admin"
        //   password = "admin"
        // }
    }
}
```

This final component creates a prometheus.remote_write component named metrics_service that points to http://localhost:9090/api/v1/write.

This completes the simple configuration pipeline.

Tip

The `basic_auth` is commented out because the local `docker compose` stack doesn’t require it. It’s included in this example to show how you can configure authorization for other environments.

For further authorization options, refer to the `prometheus.remote_write` component documentation.

This connects directly to the Prometheus instance running in the Docker container.

## Reload the configuration

Copy your local `config.alloy` file into the default Alloy configuration file location.

```java
sudo cp config.alloy /etc/alloy/config.alloy
```

Call the /-/reload endpoint to tell Alloy to reload the configuration file without a system service restart.

```java
curl -X POST http://localhost:12345/-/reload
```

Tip

This step uses the Alloy UI, on `localhost` port `12345`. If you choose to run Alloy in a Docker container, make sure you use the `--server.http.listen-addr=0.0.0.0:12345` argument.

If you don’t use this argument, the debugging UI won’t be available outside of the Docker container.

Optional: You can do a system service restart Alloy and load the configuration file:

```java
sudo systemctl reload alloy

sudo systemctl status alloy

# if it fails to start, review the logs:

journalctl -u alloy.service -n 50

```

## Inspect your configuration in the Alloy UI

Open http://localhost:12345 and click the Graph tab at the top. The graph should look similar to the following:

Your configuration in the Alloy UI
![image](/images/Metrics-inspect-your-config.png)

The Alloy UI shows you a visual representation of the pipeline you built with your Alloy component configuration.

You can see that the components are healthy, and you are ready to explore the metrics in Grafana.

## Log into Grafana and explore metrics in Prometheus

Open http://localhost:3000/explore/metrics/ to access the Explore Metrics feature in Grafana.

From here you can visually explore the metrics sent to Prometheus by Alloy.

Explore Metrics App
![image](/images/explore-metrics.png)

You can also build PromQL queries manually to explore the data further.

Open http://localhost:3000/explore to access the Explore feature in Grafana.

Select Prometheus as the data source and click the Metrics Browser button to select the metric, labels, and values for your labels.

Here you can see that metrics are flowing through to Prometheus as expected, and the end-to-end configuration was successful.

### Summary

You have configured Alloy to collect and process metrics from your local host and send them to your local Grafana stack.
