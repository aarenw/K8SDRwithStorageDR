
# Sample script

```
AWS_KEY=$(oc get secret dc1bucket -n dc1 -o jsonpath='{.data.AWS_ACCESS_KEY_ID}' | base64 -d)
AWS_SECRET=$(oc get secret dc1bucket -n dc1 -o jsonpath='{.data.AWS_SECRET_ACCESS_KEY}' | base64 -d)
BUCKET_NAME=$(oc get obc dc1bucket -n dc1 -o jsonpath='{.spec.bucketName}')

TMPFILE=$(mktemp)
cat > "$TMPFILE" <<EOF
[default]
aws_access_key_id=${AWS_KEY}
aws_secret_access_key=${AWS_SECRET}
EOF

oc create secret generic cloud-credentials -n openshift-adp --from-file=cloud="$TMPFILE" --dry-run=client -o yaml | oc apply -f -
rm -f "$TMPFILE"

cat <<EOF | oc apply -f -
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: velero-instance
  namespace: openshift-adp
spec:
  backupLocations:
    - name: default
      velero:
        provider: aws
        default: true
        objectStorage:
          bucket: ${BUCKET_NAME}
          prefix: velero
        config:
          region: us-east-1
          profile: default
          s3ForcePathStyle: "true"
          s3Url: https://s3.openshift-storage.svc
          insecureSkipTLSVerify: "true"
        credential:
          name: cloud-credentials
          key: cloud
  configuration:
    nodeAgent:
      enable: true
      uploaderType: kopia
    velero:
      defaultPlugins:
        - openshift
        - aws
        - csi
        - kubevirt
      featureFlags:
        - EnableCSI
  logFormat: text
EOF
```