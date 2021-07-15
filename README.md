# openshift-vsftpd

I started installing fedora with vsftpd. Then thought ...

Lets use https://github.com/atmoz/sftp which has 200m downloads - not bad.

I whole-heartedly agree with this blog post, that [FtpMustDie.](https://mywiki.wooledge.org/FtpMustDie)

Your mileage will vary.

## Installation

Basic installation:
```bash
oc new-project sftp --display-name="Internal sftp server"
oc new-app atmoz/sftp:alpine
oc apply -f anyuid-sftp.yaml
oc adm policy add-scc-to-user anyuid-sftp -z default
oc create configmap users-config --from-file=users.conf
oc set volume deployment/sftp --add --overwrite -t configmap --configmap-name=users-config --name=users-config --mount-path=/etc/sftp/users.conf --sub-path=users.conf --overwrite --default-mode='0755' --read-only=false
ssh-keygen -t ed25519 -f ssh_host_ed25519_key < /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null
oc create configmap ssh-config --from-file=sshd_config --from-file=ssh_host_ed25519_key --from-file=ssh_host_ed25519_key.pub --from-file=ssh_host_rsa_key --from-file=ssh_host_rsa_key.pub
oc set volume deployment/sftp --add --overwrite -t configmap --configmap-name=ssh-config --name=ssh-config --mount-path=/etc/ssh --overwrite --default-mode='0600' --read-only=true
```

Add persistent storage:
```bash
cat <<'EOF' | oc apply -f-
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sftp
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  volumeMode: Filesystem
EOF
oc set volume deployment/sftp --add --overwrite -t persistentVolumeClaim --claim-name=sftp --name=sftp --mount-path=/home
```

SNI is a TLS extension, SFTP only supports SSH so you must use `NodePort` or similar to expose sftp outside the cluster. Port Forward also works for testing use cases. 
## Using

Command line:
```bash
oc port-forward svc/sftp 22:22

$ sftp foo@localhost
foo@localhost's password: 
Connected to localhost.
sftp> ls
upload
```

If you get this error:
```bash
Too many authentication failures [preauth]
```
You may need to set this:
```bash
~/.ssh/config
Host * 
       	IdentitiesOnly=yes
```

Filezilla also works. Note you cannot disable use of `pagent`, or make use of `IndentitiesOnly` with Fiezilla. So if you have lots of keys loaded make sure this ssh_config setting is large enough `MaxAuthTries 20`.