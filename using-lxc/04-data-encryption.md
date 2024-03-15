# Generating the Data Encryption Config and Key

Kubernetes stores a variety of data including cluster state, application configurations, and secrets. Kubernetes supports the ability to encrypt cluster data at rest.

In this we will be generate an encryption key and an encryption config suitable for encrypting Kubernetes Secrets.

## The Encryption Key

Generate an encryption key:

    ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)


## The Encryption Config File
Create the encryption-config.yaml encryption config file:

    cat > encryption-config.yaml <<EOF
    kind: EncryptionConfig
    apiVersion: v1
    resources:
      - resources:
          - secrets
        providers:
          - aescbc:
              keys:
                - name: key1
                  secret: ${ENCRYPTION_KEY}
          - identity: {}
    EOF

![alt text](image-17.png)

### Copy the encryption-config.yaml encryption config file to each controller instance:


    for instance in controller-0 controller-1 controller-2; do
    lxc file push encryption-config.yaml ${instance}/root/
    done

![alt text](image-18.png)