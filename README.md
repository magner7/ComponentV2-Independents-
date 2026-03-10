# 🫧 Components V2 com TypeScript

Um guia simples e direto mostrando como usar Components V2 no Discord com TypeScript. A ideia aqui é mostrar uma base limpa, fácil de entender e que você pode usar em bots maiores sem virar bagunça.


---

# Sobre Components V2

Os Components V2 permitem montar interfaces mais organizadas dentro do Discord usando botões, menus e modais.

Na prática, isso ajuda muito quando você precisa criar painéis interativos, sistemas de configuração ou dashboards dentro do próprio Discord.

Com uma boa estrutura em TypeScript fica muito mais fácil manter o projeto organizado.


---

# Estrutura simples de projeto

Uma estrutura básica que já ajuda bastante a manter tudo organizado:

```js
src/
 client/
 components/
 events/
 utils/
 index.ts
```
Separar as coisas assim evita arquivos gigantes e deixa o código muito mais fácil de manter.


---

# Tipagem de customId

Uma boa prática é padronizar os customId. Isso evita erro e deixa o código mais previsível.

```js
export type CustomIdData = {
 action: string
 userId?: string
 page?: number
}

export function encodeCustomId(data: CustomIdData) {
 return JSON.stringify(data)
}

export function decodeCustomId(id: string): CustomIdData | null {
 try {
  return JSON.parse(id)
 } catch {
  return null
 }
}
```

Isso ajuda bastante quando o bot começa a crescer.


---

# Criando o client

```js
import { Client, GatewayIntentBits } from "discord.js"

export function createClient(){
 return new Client({
  intents:[GatewayIntentBits.Guilds]
 })
}
```


---

# Arquivo principal
```js
import "dotenv/config"
import { createClient } from "./client/createClient"

const client = createClient()

client.once("clientReady",()=>{
 console.log(`Bot online ${client.user?.tag}`)
})

client.login(process.env.DISCORD_TOKEN)
```

---

# Criando um painel com botão e select

**Aqui está um exemplo simples de painel usando botões e menu de seleção.**

```js
import {
 ActionRowBuilder,
 ButtonBuilder,
 ButtonStyle,
 StringSelectMenuBuilder,
 EmbedBuilder
} from "discord.js"

export function buildPanel(){

 const embed = new EmbedBuilder()
 .setTitle("Painel")
 .setDescription("Exemplo simples usando Components V2")

 const button = new ButtonBuilder()
 .setCustomId("config")
 .setLabel("Configurações")
 .setStyle(ButtonStyle.Primary)

 const select = new StringSelectMenuBuilder()
 .setCustomId("menu")
 .setPlaceholder("Escolha um módulo")
 .addOptions(
  {label:"Tickets",value:"tickets"},
  {label:"Logs",value:"logs"},
  {label:"Moderação",value:"mod"}
 )

 const rowButtons = new ActionRowBuilder<ButtonBuilder>()
 .addComponents(button)

 const rowMenu = new ActionRowBuilder<StringSelectMenuBuilder>()
 .addComponents(select)

 return {
  embeds:[embed],
  components:[rowButtons,rowMenu]
 }
}
```



---

# interactionCreate

Aqui é onde o bot escuta os componentes sendo usados.

```js
client.on("interactionCreate",async interaction=>{

 if(interaction.isButton()){

  if(interaction.customId === "config"){
   await interaction.reply({
    content:"Abrindo configuração",
    ephemeral:true
   })
  }

 }

 if(interaction.isStringSelectMenu()){

  const value = interaction.values[0]

  if(value === "tickets"){
   await interaction.reply({content:"Sistema de tickets"})
  }

  if(value === "logs"){
   await interaction.reply({content:"Sistema de logs"})
  }

 }

})
```

---

# Enviando o painel

```js
const panel = buildPanel()

interaction.reply(panel)
```

---

# Algumas boas práticas

- Separe handlers sempre que possível
- Evite colocar lógica grande dentro de eventos
- Padronize customId
- Use TypeScript para evitar erros comuns
