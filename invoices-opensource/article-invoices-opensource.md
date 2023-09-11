# How to Deploy Dolibarr on Kubernetes - ERP & CRM System

Ever wondered how to create invoices without relying on paid services or abusing Microsoft Excel? [Dolibarr](https://www.dolibarr.org/) is a software that provides functionality to create invoices and offers, as well as to manage corresponding resources such as customers, partners, products, service, projects, payments, etc. It is free and open-source. After setup, You can access your own Dolibarr instance via web browser. Officially, Dolibarr is referred as an ERP (Enterprise Resource Planning) and CRM (Customer Relationship Management) system. From my experience, it is overall the best choice among open-source systems. Other candidates are [Odoo](https://github.com/odoo/odoo), [Metafresh](https://github.com/metasfresh/metasfresh), [Kivitendo](https://github.com/kivitendo/kivitendo-erp), and [Apache OFBiz](https://github.com/apache/ofbiz-framework).

This article shows how to set up Dolibarr in your Kubernetes cluster. This way you can benefit from Kubernetes advantages such as high availability and automatic volume backups (if enabled). Although, this might be overkill for some uses cases. As a bonus, a reference to a simple setup using Docker compose is also included.

Try out a Dolibarr demo at https://demo.dolibarr.org

![Screenshot of a Dolibarr page to edit an invoice](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/d0xs7sj3zly0zszvj069.png)

## Prerequisites

- Kubernetes cluster is set up and ready
- Subdomain e.g. `dolibarr.example.com` points to your Kubernetes cluster
- Kubernetes cluster has a `ClusterIssuer` named `lets-encrypt` set up and ready (you can adjust this setup if named differently)


## Deployment @ Kubernetes

To deploy Dolibarr on your Kubernetes cluster, multiple Kubernetes objects are needed. The GitHub repository https://github.com/soerenmetje/kubernetes-dolibarr  contains Kubernetes template files (`*.yaml`) and detailed setup instructions. In summary, following steps are necessary:
1. Create a new namespace
2. Modify the template files
3. Apply the template files to create the Kubernetes objects
4. Wait
5. Check if Dolibarr instance is reachable

If you want to restrict the access to certain IPs or IP ranges, further configuration steps are included in the repository. Additionally, the upgrading process and advanced configuration of Dolibarr, as well as known issues and troubleshooting are covered.

### Architecture

In the end, we will end up with following Kubernetes objects:

![Dolibarr Kubernetes architecture](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/apswyqov3w86hdqr1aho.png)

In the `webserver` pod, [Tuxgasys' Dolibarr Docker image](https://hub.docker.com/r/tuxgasy/dolibarr) is used. Luckily, this image handles the installation process of Dolibarr automatically on the first start. The `db` pod runs a [mariadb](https://hub.docker.com/_/mariadb) container. The persistent volumes `dolibarr-custom`, `dolibarr-documents`, and `db-data` ensure that configurations, generated PDF files, and the database content is persistently stored. The `dolibarr-ingress` ingress allows accessing Dolibarr via https using a user-defined subdomain. 

## Bonus: Deployment @ Docker

In the GitHub repository of Tuxgasys' Dolibarr Docker image a [Docker compose deployment](https://github.com/tuxgasy/docker-dolibarr/tree/master/examples/with-rp-traefik) is included that integrates [traefik](https://hub.docker.com/_/traefik) as a reverse proxy.

## Sources

- https://github.com/tuxgasy/docker-dolibarr
- https://github.com/Dolibarr/dolibarr
- https://wiki.dolibarr.org/index.php?title=Home
- https://github.com/soerenmetje/kubernetes-dolibarr
