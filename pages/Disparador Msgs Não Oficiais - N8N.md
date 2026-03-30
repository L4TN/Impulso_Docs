Build Chatwoot
Remover templates Oficiais da Tela de Agendamentos do Chatwoot (Dias)/Remover complexidade
 Cria tabela e insert de Campanha na Tb_Campanha (Nome, Periodicidade/Cron, Mensagem. Vars, Inbox, Link Anexo, Lista Contatos, Tipo de Midia) 
FrontEnd enviar Request e Salvar linha na Tb_Campanha

Request:
{
  "nome": "name_campanha",
  "mensagem": "Olá {1}, tudo bem ? Aqui é a {2}",
  "lista_contato": [
    {
      "numero": "5515991880219",
      "variaveis": {
        "1": "Sérgio",
        "2": "ImpulsoCore"
      }
    },
    {
      "numero": "5515988888888",
      "variaveis": {
        "1": "Maria",
        "2": "Empresa Y"
      }
    }
  ],
  "dt_inicio": "2026-01-18T12:00:00",
  "dt_fim": "2026-01-19T21:23:59",
  "dias_semana": [
    "Sun",
    "Mon",
    "Tue",
    "Wed",
    "Thu",
    "Fri",
    "Sat"
  ],
  "media_type": "image",
  "mime_type": "image/jpeg",
  "link_media": "https://picsum.photos/600/400.jpg",
  "account":"6", 
  "inbox": "10"
}

campanha

id, nome, mensagem, lista_contato, dt_inicio, dt_fim, dias_semana, fl_ativo, account, inbox 

Criar fluxo N8N para Disparador Evolution
- Crir um "Schedule Trigger" com periodicidade de 1min
- Conectar no banco de dados Postgres "infra_iara" e bater na tabela campanha filtrando:
    **Onde** o **Date Time Now** está dentro de dt_inicio e dt_fim **E** onde o **fl_ativo** igual a **True** 
    -> Result de N linhas chamado Lista_Campanhas
    -> Para cada item de Lista_Campanhas
        **SE** Lista_Campanhas.dias_semana contém o **Actual Day**
        **Então** Para cada item de Lista_Campanhas.lista_contato
            number é igual a item.numero + "@s.whatsapp.net"
            *mensagem é igual a mensagem com os placeholders substituidos
            caption é igual a mensagem 
            fileName é igual "imagem" + "{extension}"
            account: 
            inbox:

            endpoint = Pegue account e inbox e filtre na tabela aplicacao_canal onde Account = account e Inbox = inbox e selecione instancia Evolution

        Dispara Evolution (Tem exemplo no 6. Agente e Ferramentas IARA)
        curl -L "http://38.242.214.250:8080/message/sendMedia/{endpoint}" -H "apikey: AHAIKD@O99834" -H "Content-Type: application/json" --data-raw 
        "{
            \"number\": \"{number}@s.whatsapp.net\",
            \"mediatype\": \"image\",
            \"mimetype\": \"image/jpeg\",
            \"caption\": \"Posso explicar rapidinho como ajudamos empresas?\",
            \"media\": \"https://picsum.photos/600/400.jpg\",
            \"fileName\": \"foto.jpg\"
        }"

    -> Pra cada linha


Disparar usando exemplo de Media(Postman) e exemplo da Iara 



