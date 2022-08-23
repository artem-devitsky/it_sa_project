# it-sa-project-private

## bash_history
```bash
pip install requests google-auth
openssl genrsa -out master-it-sa.key 4096
openssl req -x509 -new -key master-it-sa.key -out master-it-sa.pem -days 365
ansible-galaxy collection install google.cloud
ansible-galaxy collection install community.general
ansible-galaxy collection install ansible.utils
ansible-galaxy collection install ansible.posix
pip install netaddr
```
https://github.com/nagaraj1171/WordPress-CI-CD-K8/blob/master/Jenkinsfile

```bash
jsonpath="{.data.jenkins-admin-password}"
secret=$(kubectl get secret -n jenkins jenkins -o jsonpath=$jsonpath)
echo $(echo $secret | base64 --decode)
```

https://devopscube.com/jenkins-build-agents-kubernetes/

```bash
install:
container-diff @ google #shit
kubeval
```
sudo git clone https://github.com/moul/docker-diff /opt/docker-diff 
sudo ln -s /opt/docker-diff/docker-diff /usr/local/bin

test webhook string