# How to Connect Nextcloud and Collabora


This article guides you through the process of connecting two separate containers: 1. Nextcloud  2. Collabora. 
It explains how to set up a connection between the two, allowing Nextcloud to open and edit documents using the Collabora online document editor.

## Introduction

**Nextcloud** is primarily designed as a file-sharing system and does not offer advanced document editing capabilities. **Collabora**, on the other hand, is a standalone application for collaborative document editing. By integrating Collabora with Nextcloud, you can leverage its editing features for documents stored in Nextcloud.

> Note: While it's possible to install Collabora Code as a plugin within Nextcloud itself (using the **"Collabora Online - Built-in CODE Server"** app available at [https://apps.nextcloud.com/apps/richdocumentscode](https://apps.nextcloud.com/apps/richdocumentscode)), this is not recommended for production environments. This article focuses on an independent installation of Collabora, followed by connecting it to Nextcloud.

# Nextcloud
> for installing **nextcloud** on the air-gapped openshift cluster, you will need to use a spechile image that wase rebild whit a uniq UID. for more infomation on this, see this guide: *[linnovate wiki link]*
To configure Nextcloud to work with Collabora, follow these steps:
1. connect to nextcloud with the admin user.
2.  In Nextcloud, navigate to the "Apps" section and install the "Nextcloud Office" app. You can manually download the app and copy its files to the apps directory in Nextcloud. The app can be downloaded from: [https://apps.nextcloud.com/apps/richdocuments](https://apps.nextcloud.com/apps/richdocuments)
    
3.  After installing the app, go to the right upper circle and go to **"administrator settings"**. on the left menu bar go to **"administrator"** section, and then to **"nextcloud office"**
![screenshot_from_2023-05-17_12-58-05.png](/screenshot_from_2023-05-17_12-58-05.png)
![screenshot_from_2023-05-17_13-07-08.png](/screenshot_from_2023-05-17_13-07-08.png)
![screenshot_from_2023-05-17_13-08-15.png](/screenshot_from_2023-05-17_13-08-15.png)
    
4.  Configure the Collabora URL in the app settings. This URL should point to your Collabora server installation. Save the changes.

5. add this varialbe to the deployment
`extra_parms= --o:storage.webdav.url=<your-nextlcoud-full-url>/remote.php/webdav/`

# Collabora

To allow incoming traffic from Nextcloud and establish the connection, follow these steps:

1.  Install and configure Collabora on a separate server or container. 
> for installing **Collabora** on a air-gapped openshift cluster, (and for more general infomation on the collabora instaltion as well)  you will need to follow this guide: *[linnovate wiki link]*
    
2. to allow incoming trafic from nextcloud, add this variable to the collabora installation:

`aliasgroup1=https://<nextcloud-domain>`
replace `<nextcloud-domain>` with the URL of your nextcloud installation 
 the value of this variable can be writen in different ways. see here: https://sdk.collaboraonline.com/docs/installation/CODE_Docker_image.html

3. add these variable to the collabora instalation:
`extra_params=--o:security.capabilities=false --o:ssl.enable=false --o:ssl:termination=true --o:net.proto:IPv4`  

if you want to access the dashboard Collabora, add these variables:
`username=admin`
`password=1234`

the collabora dashboard is accesble via this url:
`https://<collabora.example.com:9980>/loleaflet/dist/admin/admin.html`

By following these steps, you can establish a connection between Nextcloud and Collabora, enabling seamless document editing within Nextcloud using the powerful features of Collabora.

