##### Private Registry for Docker Image with Kubernetes & Only Allowed Access from `docker login` or CLI, Otherwise Will Result in 403 Forbidden if Accessed from Browser

## Prerequisites
```zsh
registry-kubernetes@febrd:~$ sudo apt install apache2-utils
```

**Start Minikube:**
```zsh
registry-kubernetes@febrd:~$ minikube start 
```
**Create SSL Name Space**
```zsh
#Example cert-manager
registry-kubernetes@febrd:~$ kubectl create namespace <your-name-space> 
```
**Navigate to Your Project Path:**
```zsh
registry-kubernetes@febrd:~/master$
```

**Create Authentication Credentials:**
```zsh
registry-kubernetes@febrd:~/master$ htpasswd -Bbn YOUR_DESIRED_USERNAME YOUR_DESIRED_PASSWORD > path/to/authentication_file/authentication_file_name
```

**Apply `regcred` Secret:**
```zsh
registry-kubernetes@febrd:~$ kubectl create secret generic regcred --from-file=htpasswd=path/to/authentication_file/authentication_file_name
```

**Enable Ingress Addon:**
```zsh
registry-kubernetes@febrd:~/master$ minikube addons enable ingress
```

## Deployment

**Create or Edit YAML Files:**
```zsh
registry-kubernetes@febrd:~/master$ sudo vi registry-deployments.yaml
registry-kubernetes@febrd:~/master$ sudo vi registry-service.yaml
registry-kubernetes@febrd:~/master$ sudo vi ingress.yaml
registry-kubernetes@febrd:~/master$ sudo vi cluster-issuer.yaml
```

**Create Directory for Registry Storage:**
```zsh
registry-kubernetes@febrd:~/master$ mkdir /path/to/desired/folder_name
```

**Apply YAML Configurations:**
```zsh
#Deploy Registry
registry-kubernetes@febrd:~/master$ kubectl apply -f registry-deployments.yaml

# Apply Registry Service
registry-kubernetes@febrd:~/master$ kubectl apply -f registry-service.yaml

# Apply Ingress
registry-kubernetes@febrd:~/master$ kubectl apply -f ingress.yaml

# Apply SSL Certificates
registry-kubernetes@febrd:~/master$ kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.7.1/cert-manager.yaml

# Apply Cluster Issuer
registry-kubernetes@febrd:~/master$ kubectl apply -f cluster-issuer.yaml
```

## Troubleshooting

**Check Pods Status:**
```zsh
registry-kubernetes@febrd:~/master$ kubectl get pods
```
### Deployment Error:
**Cluster Issuer**
```
#Wait Untill cert-manager status are running
registry-kubernetes@febrd:~/master$ kubectl get pods --namespace <your-cert-manager-name-space> #Example cert-manager
```
### HTTP Error Code
**Port 8080: Refused**
```
Don't use sudo when running kubectl commands.
```
**Forbidden 401**
```
Wrong Credentials
```
**Forbidden 403**
```
Wrong access request origin. Try using CLI or Docker commands.
```
**/V2/ Not Found 404**
```
DNS is not pointing to the correct server in the configuration that was created.
```
**Bad Gateway 502**
```
Check Configuration for DNS & Ingress
```


## Configuration

**Configure Docker Daemon for HTTP Support:**
```zsh
registry-kubernetes@febrd:~/master$ mkdir /path/to/file/daemon.json
```

**Edit `daemon.json` for Insecure Registries:**
```json
{
  "insecure-registries": ["your_kubernetes_IP:YourNodePort"]
}
```

**Identify NodePort for Docker Access:**
```zsh
registry-kubernetes@febrd:~/master$ kubectl get svc registry-service
```

## Testing

**Test Docker Login - HTTP:**
```zsh
registry-kubernetes@febrd:~/master$ docker login http://YourKubernetesIP:NodePort
```

**Test Docker Login - HTTPS:**
```zsh
registry-kubernetes@febrd:~/master$ docker login https://YourKubernetesIP:NodePort
```

## Setup DNS

**Configure Nginx for Domain Routing:**
```zsh
registry-kubernetes@febrd:~/master$ sudo vim /etc/nginx/sites-available/your.domain.name
```

```nginx
server {
    server_name your.domain.name;

    location /v2/ {
        proxy_pass http://YOUR-KUBERNETES_IP:YourNodePort/v2/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        if ($http_user_agent ~* (mozilla|chrome|safari)) {
            return 403;
        }

        satisfy any;
        allow 127.0.0.1;  # Allow access from localhost
    }

    location /v2/_catalog {
        auth_basic "Restricted Content";
        auth_basic_user_file /path/to/authentication_file;
        proxy_pass http://YOUR-KUBERNETES_IP:YourNodePort/v2/;
    }

    location / {
        proxy_pass http://YOUR-KUBERNETES_IP:YourNodePort;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```
**Create Symlink:**
```zsh
registry-kubernetes@febrd:~/master$ sudo ln -s /path/to/config_nginx/your.domain.name /desired/path/to/nginx/enabled
```

**Validate Nginx Configuration:**
```zsh
registry-kubernetes@febrd:~/master$ sudo nginx -t
```

**Reload Nginx:**
```zsh
registry-kubernetes@febrd:~/master$ sudo systemctl reload nginx
```

## Final Testing

**Test Docker Login with Domain/Subdomain:**
```zsh
┌──(client㉿outside)-[~/demo]
└─$ docker login https://your.domain.name
```

## Conclusion

Congratulations! Your private Docker registry is set up and ready for use !.
