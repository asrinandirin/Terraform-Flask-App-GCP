## Basic Flask Web Server with Terraform on GCP 
___
This is a basic infrastructure to understand Terraform basics.

In real-life scenarios, you will face different configuration architectures (not all configurations in a single main.tf file).

This code includes a backend configuration to store the state file in GCP Storage Buckets. Make sure you have a bucket and it is properly configured in the first lines of main.tf.

___
###  Backend config to store your state file safely.

    terraform {
    backend "gcs" {
    bucket  = "YOUR-BUCKET-NAME"
    prefix  = "terraform/state"
    }
    }

___
### 1 - Create your VPC and Subnet.

    resource "google_compute_network" "vpc_network" {
    name                    = "my-custom-VPC"
    auto_create_subnetworks = false
    mtu                     = 1460
    }

    resource "google_compute_subnetwork" "default" {
    name          = "my-custom-subnet"
    ip_cidr_range = "10.0.1.0/24"
    region        = "us-west1"
    network       = google_compute_network.vpc_network.id
    }
___

### 2 - Create Compute Engine Instance
    resource "google_compute_instance" "default" {
    name         = "flask-vm"
    machine_type = "f1-micro"
    zone         = "us-west1-a"
    tags         = ["ssh"]

    boot_disk {
        initialize_params {
        image = "debian-cloud/debian-11"
        }
    }

    // Install Flask
    metadata_startup_script = "sudo apt-get update; sudo apt-get install -yq build-essential python3-pip rsync; pip install flask"

    network_interface {
        subnetwork = google_compute_subnetwork.default.id
    }
    }
___ 

### 3 - Create SSH Rule to be able to connect VM 

    resource "google_compute_firewall" "ssh" {
    name = "allow-ssh"
    allow {
        ports    = ["22"]
        protocol = "tcp"
    }
    direction     = "INGRESS"
    network       = google_compute_network.vpc_network.id
    priority      = 1000
    source_ranges = ["0.0.0.0/0"]
    target_tags   = ["ssh"]
    }

___

### 4 - (Optional) To be able to access server from your local environment, 
    resource "google_compute_firewall" "flask" {
    name    = "flask-app-firewall"
    network = google_compute_network.vpc_network.id

    allow {
        protocol = "tcp"
        ports    = ["5000"]
    }
    source_ranges = ["0.0.0.0/0"]
    }

___
 
### 5 - (Optional) A variable for extracting the external IP address of the VM

    output "Web-server-URL" {
    value = join("",["http://",google_compute_instance.default.network_interface.0.access_config.0.nat_ip,":5000"])
    }


___
### Apply your configuration to your environment (you must know what each command does up to this point). 

    Terraform init 
    Terraform plan
    Terraform apply 
___

### After creating all resources in your GCP environment, connect vm via ssh and make changes below

    vi app.py

___
### Inside app.py, 

    from flask import Flask
    app = Flask(__name__)

    @app.route('/')
    def hello_cloud():
    return 'Hello Cloud!'

    app.run(host='0.0.0.0')
___

### Then run your app, 
    python3 app.py

___

