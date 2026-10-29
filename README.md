# Telemetry pipeline setup

## On premises
### Set up Raspberry PI
1. Connect to WiFi if not configured during imaging
```
sudo nmcli connection add type wifi con-name EdelbyteGuest ssid EdelbyteGuest wifi-sec.key-mgmt wpa-psk wifi-sec.psk <password>
sudo nmcli connection up EdelbyteGuest
```
2. Install mosquitto, vector
```bash
sudo apt install mosquitto mosquitto-clients
bash -c "$(curl -L https://setup.vector.dev)"
sudo apt update
sudo apt install vector
```
#### Setup vector
vector validate does not support secret substitutions, we need to configure vector service accordingly
```
sudo systemctl edit vector
```
Paste this into the editor that opens (a drop-in file, not the original unit):

```
[Service]
ExecStartPre=
ExecStartPre=/usr/bin/vector validate --skip-healthchecks

ExecReload=
ExecReload=/usr/bin/vector validate --no-environment --skip-healthchecks
ExecReload=/bin/kill -HUP $MAINPID
```

The empty ExecStartPre=/ExecReload= lines are required — systemd appends to Exec*= directives in drop-ins by default, so without clearing them first you'd end up with both the old (broken) and new validate calls running.

Then:

```
sudo systemctl daemon-reload
sudo systemctl restart vector
sudo journalctl -u vector -f
```

2. Configure static IP assignment on DHCP server
    - Wired on office network
    - Wi-Fi on guest network
3. Deploy vector-office.yaml to /etc/vector/vector.yaml
4. Create /etc/vector/s3.secrets - json - those are not really secrets just bucket name and region 
```
{
  "region": "<...>",
  "telemetry_bucket": "<...>"
}
```
5. Create /etc/vector/aws.credentials
```
[default]
aws_access_key_id = AKIA<...>
aws_secret_access_key = <...>
```
6. Create /etc/vector/mqtt.secrets - those must match mosquitto configration and be deployed to the sensors as well
```
{
  "user": "<mqtt user>",
  "password": "<mqtt password>"
}
```
#### Setup mosquitto
```
sudo mosquitto_passwd -c /etc/mosquitto/passwd <mqtt user>
# prompts for a password — enter the same value as mqtt_password in secrets.yaml
sudo chown mosquitto:mosquitto /etc/mosquitto/passwd
sudo chmod 640 /etc/mosquitto/passwd


```

## In AWS
### Create bucket and access for Raspberry PI to upload telemetry
In theory nothing blocks you from accessing AWS resources from Raspberry PI, but I would refrain from storing access tokens with account admin permissions on this device. 
So run this on your laptop, following commands are for Windows:
Names that will be used in the setup
```
set AWS_PROFILE=myprofile
set REGION=eu-north-1
set BUCKET=edelbyte-office-telemetry
set IAM_USER=edelbyte-office-telemetry-uploader
set IAM_POLICY=edelbyte-office-telemetry-upload-permissions

aws s3api create-bucket --bucket %BUCKET% --region %REGION% --create-bucket-configuration LocationConstraint=%REGION%

aws iam create-user --user-name %IAM_USER%

(
echo {
echo   "Version": "2012-10-17",
echo   "Statement": [
echo     {
echo       "Sid": "AllowPutTelemetryObjects",
echo       "Effect": "Allow",
echo       "Action": ["s3:PutObject", "s3:AbortMultipartUpload"],
echo       "Resource": "arn:aws:s3:::%BUCKET%/*"
echo     },
echo     {
echo       "Sid": "ListBucket",
echo       "Effect": "Allow",
echo       "Action": ["s3:ListBucket"],
echo       "Resource": "arn:aws:s3:::%BUCKET%"
echo      }
echo   ]
echo }
) > %IAM_POLICY%.json

aws iam put-user-policy --user-name %IAM_USER% --policy-name %IAM_POLICY% --policy-document file://%IAM_POLICY%.json
```
