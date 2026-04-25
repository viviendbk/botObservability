


log format : 

  "records": [
    {
      "key": "bot-001",
      "value": {
        "botName": "bot-001",
        "server": "Boune",
        "kamasBruts": 15000,
        "kamasBank": 450000,
        "kamasHdv": 30000
      }
    }
  ]


  respecter la key car ça permet au bridge d'envoyer les messages sur la bonne partition du topic


  # Get your Minikube IP
export MINIKUBE_IP=$(minikube ip)

# Send an HTTP POST to the Bridge
curl -X POST \
  http://$MINIKUBE_IP:32080/topics/bot_events \
  -H 'Content-Type: application/vnd.kafka.json.v2+json' \
  -d '{
  "records": [
    {
      "key": "bot-001",
      "value": {
        "botName": "bot-001",
        "server": "Boune",
        "kamasBruts": 15000,
        "kamasBank": 450000,
        "kamasHdv": 30000
      }
    }
  ]
  }'