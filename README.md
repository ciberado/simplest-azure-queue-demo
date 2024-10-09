# Simple Azure Storage Acccount Queue Demo

We are going to create an Azure Storage Account and a queue within it. This setup will be used for a simple demonstration of a producer-consumer scenario. The producer will generate messages and put them into the queue. The consumer will retrieve these messages and delete them from the queue after reading. 

Let's define a unique prefix so we don't collision at resource naming.

```bash
MYPREFIX=<...>
```

And now we can create a Resource Group for our demo:

```bash
az group create \
  --name $MYPREFIX-rg \
  --location westeurope
```

The queue resides inside a Storage Account, so we need to create it:

```bash
az storage account create \
  --name ${MYPREFIX}queue \
  --resource-group $MYPREFIX-rg \
  --sku Standard_LRS \
  --encryption-services blob
```

Next, we are creating a new queue named 'demo' within the storage account we just created. This queue will be used to store the messages produced by our producer script.

```bash
az storage queue create \
  --account-name ${MYPREFIX}queue \
  --name demo
```

The following producer will put a message in the queue each second. Take a look at it for understanding how the process goes, although is pretty direct.

```bash
cat << 'EOF' > producer.sh
#!/bin/sh
 
MYPREFIX=$1
echo Using queue demo.
for i in `seq 1 1000`; do
  az storage message put \
    --account-name ${MYPREFIX}queue \
    --queue-name demo \
    --content "This is message $i." \
    2>/dev/null
  sleep 1
done
EOF
```

The consumer reads the queue at a slower rate, which is nice to see how the queue is acting as a buffer. Also, it is important to remember that messages must be explicitly delete after processing them, or they will appear in the queue again after the visibility timeout.

```bash
cat << 'EOF' > consumer.sh
#!/bin/sh

MYPREFIX=$1

for i in `seq 1 1000`; do

  RESPONSE=$(az storage message get \
    --account-name ${MYPREFIX}queue \
    --queue-name demo \
    --visibility-timeout 30 \
    2>/dev/null)
  
  if [[ "${RESPONSE}" = "[]" ]]; then
    echo "No messages in the queue.";
  else
    CONTENT=$(echo $RESPONSE | jq ".[].content" -r)
    ID=$(echo $RESPONSE | jq ".[].id" -r)
    POP=$(echo $RESPONSE | jq ".[].popReceipt" -r)

    echo The message says: $CONTENT

    az storage message delete \
        --account-name ${MYPREFIX}queue \
        --queue-name demo \
        --id $ID \
        --pop-receipt $POP \
        2>/dev/null
  fi
  sleep 2
done
EOF
```

Finally, launch as many producers and consumers you need in new *panes*:

```bash
tmux splitw -t "$session_uuid:" -dv "bash ./producer.sh $MYPREFIX"

tmux splitw -t "$session_uuid:" -dv "bash ./consumer.sh $MYPREFIX"
```

And don't forget to delete the Resource Group once the lab is over!

```bash
az group delete \
  --name $MYPREFIX-rg \
```