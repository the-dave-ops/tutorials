# Proper environment settings for n8n self hosting

For self hosting of n8n on Kubernetes, you can use the following official GitHub repo:
https://github.com/n8n-io/n8n-hosting/blob/main/kubernetes/

But after the deployment, you will see that in the UI, the site address is http://localhost:5678, instead of the domain of the ingress.
To fix this, you need to set up some environment variables so it would work properly. The documentation on this was not so clear, so I spent some time on this issue, and this is the setup that works for me.

To the main deployment file, you need to add these variables:

```
- name: N8N_HOST
  value: n8n.example.com # replace with your domain
- name: N8N_PORT
  value: '5678'
- name: N8N_PROTOCOL
  value: https
- name: N8N_EDITOR_BASE_URL
  value: https://n8n.example.com # replace with your domain
- name: WEBHOOK_URL
  value: https://n8n.example.com/ # replace with your domain
```
using these environments variables, the UI will show the correct domain, and the webhook will work properly.

You can find the official deployment file here: https://github.com/n8n-io/n8n-hosting/blob/main/kubernetes/n8n-deployment.yaml

In this repo you can see A fixed version of the deployment file with the environment variables added.