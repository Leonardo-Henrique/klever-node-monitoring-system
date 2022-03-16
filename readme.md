# How to set up a full monitoring system to your Klever Node

Monitoring is an important aspect when dealing with servers. Knowing the level of use of your resources (memory, CPU, processes, etc.) helps to have an overview of the machine's state, which in turn, is important in its management. In addition to the information inherent to the machine itself, we need to be aware of how the Klever network is doing, as well as the status of our Node's communication with it. If you want to know how to set up a complete monitoring system on your Node, below is a detailed step-by-step tutorial.


## The whole picture

I advise you to keep things as decoupled as possible, so scaling resources or even managing your servers becomes a less arduous task. We will use the following services to configure the Node monitoring system

- Node Exporter
- Prometheus
- Grafana

Ideally, Prometheus and Node Exporter will be installed on the same machine as Node while Grafana will reside on a dedicated server just for it.

So, be sure to correctly note the IP of the Node machine as well as publicly expose the ports of the services so that you can test them and create the connection to the Grafana machine.

To do so, on the machine where Node is running, open ports 9090 (Prometheus), 9100 (Node Exporter) and 8080 (web server) in your firewall.

To verify that your Node is correctly exporting the metrics that we will use, access http://{your-ip}:8080/node/metrics (replace {your-ip} with the public IP of the machine that hosts your Node.

## Setting up the monitoring

I am running all services in Debian, but the following commands should be work in other distros as well.

#### Instaling Prometheus

In your Node server, run `sudo apt-get update && sudo apt-get upgrade -y` to upgrade the packages and then install Prometheus with `sudo apt install prometheus`.

You should see Prometheus UI accessing http://{your-ip}:9090.

#### Installing Node Exporter

Also in your Node server, we will install Node Exporter that will expose a thousands of metrics about our server that can be catched after by Grafana.

Run:

`curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest| grep browser_download_url|grep linux-amd64|cut -d '"' -f 4|wget -qi -` to download the Node Exporter package

`tar -xvf node_exporter*.tar.gz` to unzip the downloaded package

`cd  node_exporter*/` to enter in the unziped directory

`sudo cp node_exporter /usr/local/bin` to copy the Node Exporte binaries to our local bin

Now creates a new systemctl service for Node Exporter with `sudo nano /etc/systemd/system/node_exporter.service`. In the file copy the content of **node-exporter-content.service** present in this repo. Press CTRL + X and then Y to save your changes and exit.

Now, run `sudo systemctl daemon-reload`, `sudo systemctl enable node_exporter` and then `sudo systemctl start node_exporter`

You can check if the service is running correctly with `sudo systemctl status node_exporter`. If so, you can go to http://{your-ip}:9100 and you will be able to see the Node Export exporting metrics.

#### Adding Klever Node metrics and Node Exporter data into Prometheus

Now, let is add the Node Explorer and the Klever Node Metrics to be scraped for Prometheus, it will expose all the things for us to have a wonderful dashboard to monitoring the whole server.

Run` sudo nano /etc/prometheus/prometheus.yml` and add this block at final (be sure to following the identation)


      - job_name: 'node_exporter'
        static_configs:
          - targets: ['localhost:9100']
      - job_name: 'validator'
        metrics_path: '/node/metrics'
        scrape_interval: 5s
        static_configs:
         - targets: ['localhost:8080']

Reload the daemon with `sudo systemctl daemon-reload` and restart Prometheus with `sudo systemctl restart prometheus`. If you want to see if everything went well, run `sudo systemctl status prometheus` to check.

You can also go to http://{your-ip}:9090/targets and see if each service is up and being scraped by Prometheus.

#### Installing Grafana (and having a super cool monitoring dashboard)

Our goal is mantaining things decentralized, so let is go to our second sever, which will be the house of our Grafana. Remember to allow incoming access into port 3000, so please allow that in your firewall rule.

Once you are there, run 

`sudo apt install gnupg2 apt-transport-https software-properties-common wget`

`sudo wget -q -O - https://packages.grafana.com/gpg.key | apt-key add -`

`sudo echo "deb https://packages.grafana.com/oss/deb stable main" | tee -a /etc/apt/sources.list.d/grafana.list`

`sudo apt update && apt install grafana`

We have Grafana installed let is reload our daemon and start the service. So, run

`sudo systemctl daemon-reload`

`sudo systemctl enable grafana-server`

`sudo systemctl start grafana-server`

------------



##### Configuring the monitoring dashboard

Go to http://{your-grafana-server-ip}:3000 and access with* login admin and password admin* (please change the password and block new users registering).

Go to Configuration > Data Sources > New Data Source. Choose Prometheus and type the correct URL to acess the Prometheus' data. It follows the patern http://{your-promtheus-server-ip}:9090.

Click in Save & Test.

Now, final step! Let's configure the dashboard. Click in "+" icon in the side bar and select import. Now, copy the content present in [this file](https://github.com/KingStake21/Klever/blob/main/Klever-Dash1.json "this file") into the "import via panel json" field. **(special thanks for KingStake21 to make the dashboard file for the community)**

You should edit two fields

1. datasource: type the name of the source of the data (probably, if you followed the instructions I gave you, it will be Prometheus)
2. title: it will be the name of your dashboard  (this field is in the very bottom of the file)

Click in "Load" and select a UID for the dashboard, it will be essencialy your URL. Avoid spaces, choose the basic

Confirm.

Now, Go to Dashboards > Browser and select the Dashboard we just created. Now, you are able to see all metrics coming from your Node in a single panel. Congrats! You have successfully installed a full monitoring system!

Your panel will look like this

![Grafana Dashboard](https://i.imgur.com/uja8LNe.png "Grafana Dashboard")
