# 🫧 Components V2 com TypeScript

Esse guia mostra como usar Components V2 no Discord com TypeScript de um jeito simples e sem enrolação. A ideia é ter uma base limpa que você consegue encaixar em qualquer bot sem virar bagunça depois.

---

## Sobre Components V2

Basicamente os Components V2 são uma forma nova de montar mensagens no Discord. Em vez de usar embed + content como sempre foi, você monta o layout inteiro com componentes — texto, seções, botões, menus, tudo junto e organizado.

Funciona muito bem pra painéis, dashboards e qualquer coisa interativa dentro do servidor. Com TypeScript então, fica bem mais tranquilo de manter sem quebrar tudo quando o projeto começa a crescer.

> ⚠️ Precisa do discord.js `14.19.3` pra cima. Confirma com `npm list discord.js` antes de começar.

---

## Dependências

Instala tudo com esses dois comandos:

```bash
npm install discord.js@latest dotenv
npm install -D typescript ts-node @types/node
```

---

## Docs

- [Discord API — Components](https://discord.com/developers/docs/components/reference)
- [discord.js — Display Components](https://discordjs.guide/legacy/popular-topics/display-components)
- [discord.js — Docs completa](https://discord.js.org/docs)
- [Mock visual de mensagens](https://discord.builders)

---

## Estrutura do projeto

Nada muito complexo, só o suficiente pra manter as coisas no lugar:

```
src/
├── client/
├── components/
│   ├── builders/
│   └── handlers/
├── events/
├── utils/
└── index.ts
```

Separar builders de handlers já resolve 90% dos casos onde o código vira uma bagunça.

---

## Como o layout funciona no Container

Antes de sair codando vale entender uma coisa: o `ContainerBuilder` é sempre o componente raiz. Tudo que você quiser mostrar vai dentro dele. Pensa nele como a caixa que agrupa o layout inteiro.

```
ContainerBuilder               ← a caixa que segura tudo
├── TextDisplayBuilder         ← texto com markdown (substitui o content)
├── SeparatorBuilder           ← um traço pra separar as seções
├── SectionBuilder             ← texto com um botão ou imagem do lado
│   ├── TextDisplayBuilder (até 3)
│   └── ButtonBuilder ou ThumbnailBuilder
├── ActionRowBuilder           ← linha de botões ou select menus
│   ├── ButtonBuilder
│   └── StringSelectMenuBuilder
└── MediaGalleryBuilder        ← grade de até 10 imagens
```

> ⚠️ Uma coisa importante: quando você usa `IsComponentsV2`, não rola mais mandar `content`, `embeds` ou `stickers` junto. Usa o `TextDisplayBuilder` no lugar do content e tá resolvido.

---

## Tipagem de customId

Padronizar os customId é uma daquelas coisas que parece bobagem no começo mas salva muito tempo depois. Cria um encode/decode simples e usa em tudo:

```ts
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

---

## Criando o client

```ts
import { Client, GatewayIntentBits } from "discord.js"

export function createClient() {
  return new Client({
    intents: [GatewayIntentBits.Guilds]
  })
}
```

---

## Arquivo principal

```ts
import "dotenv/config"
import { createClient } from "./client/createClient"

const client = createClient()

client.once("clientReady", () => {
  console.log(`Bot online ${client.user?.tag}`)
})

client.login(process.env.DISCORD_TOKEN)
```

---

## Criando um painel com Container, botão e select

Aqui é onde a mágica acontece. Tudo montado dentro do `ContainerBuilder` — cabeçalho, seção de tickets com botão do lado, select menu e botão de configuração:

```ts
import {
  ContainerBuilder,
  TextDisplayBuilder,
  SeparatorBuilder,
  SeparatorSpacingSize,
  SectionBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  StringSelectMenuBuilder,
  MessageFlags
} from "discord.js"
import { encodeCustomId } from "./utils/customId"

export function buildPanel() {

  const header = new TextDisplayBuilder()
    .setContent("#Painel de Gerenciamento\nEscolha um módulo abaixo.")

  const separator = new SeparatorBuilder()
    .setDivider(true)
    .setSpacing(SeparatorSpacingSize.Small)

  const ticketsSection = new SectionBuilder()
    .addTextDisplayComponents(
      new TextDisplayBuilder().setContent("**Tickets**"),
      new TextDisplayBuilder().setContent("Abra e gerencie tickets de suporte.")
    )
    .setButtonAccessory(
      new ButtonBuilder()
        .setCustomId(encodeCustomId({ action: "tickets" }))
        .setLabel("Acessar")
        .setStyle(ButtonStyle.Primary)
    )

  const select = new StringSelectMenuBuilder()
    .setCustomId(encodeCustomId({ action: "menu" }))
    .setPlaceholder("Escolha um módulo")
    .addOptions(
      { label: "Tickets", value: "tickets" },
      { label: "Logs", value: "logs" },
      { label: "Moderação", value: "mod" }
    )

  const rowMenu = new ActionRowBuilder<StringSelectMenuBuilder>()
    .addComponents(select)

  const btnConfig = new ButtonBuilder()
    .setCustomId(encodeCustomId({ action: "config" }))
    .setLabel("Configurações")
    .setStyle(ButtonStyle.Secondary)

  const rowButtons = new ActionRowBuilder<ButtonBuilder>()
    .addComponents(btnConfig)

  const container = new ContainerBuilder()
    .setAccentColor(0x5865f2)
    .addTextDisplayComponents(header)
    .addSeparatorComponents(separator)
    .addSectionComponents(ticketsSection)
    .addActionRowComponents(rowMenu)
    .addActionRowComponents(rowButtons)

  return {
    components: [container],
    flags: MessageFlags.IsComponentsV2
  }
}
```

---

## Handlers separados

Em vez de jogar toda a lógica no evento, separa cada tipo de interação num arquivo próprio. Fica muito mais fácil de achar e mexer depois.

**`handlers/buttons.ts`**
```ts
import { ButtonInteraction } from "discord.js"
import { decodeCustomId } from "../utils/customId"

export async function handleButton(interaction: ButtonInteraction) {
  const data = decodeCustomId(interaction.customId)
  if (!data) return

  if (data.action === "config") {
    await interaction.reply({ content: "Abrindo configuração", flags: 64 })
  }

  if (data.action === "tickets") {
    await interaction.reply({ content: "Sistema de tickets", flags: 64 })
  }
}
```

**`handlers/selectMenus.ts`**
```ts
import { StringSelectMenuInteraction } from "discord.js"

export async function handleSelectMenu(interaction: StringSelectMenuInteraction) {
  const value = interaction.values[0]

  const respostas: Record<string, string> = {
    tickets: "Sistema de tickets",
    logs: "Sistema de logs",
    mod: "Sistema de moderação"
  }

  await interaction.reply({ content: respostas[value] ?? "Módulo não encontrado" })
}
```

---

## interactionCreate

Com os handlers separados, o evento fica limpo e só faz o roteamento:

```ts
import { Client, Events } from "discord.js"
import { handleButton } from "../handlers/buttons"
import { handleSelectMenu } from "../handlers/selectMenus"

export function registerInteractionCreate(client: Client) {
  client.on(Events.InteractionCreate, async (interaction) => {
    if (interaction.isButton()) {
      await handleButton(interaction)
    }

    if (interaction.isStringSelectMenu()) {
      await handleSelectMenu(interaction)
    }
  })
}
```

---

## Enviando o painel

```ts
const panel = buildPanel()

interaction.reply(panel)
```

---

## Algumas boas práticas

- Separe handlers sempre que possível, evento limpo é evento feliz
- Não coloca lógica pesada dentro de eventos, delega pra funções
- Padroniza os customId com encode/decode desde o começo
- TypeScript tá aí pra ajudar, deixa ele reclamar antes de você descobrir o bug em prod
- Respeita os limites do Discord: no máximo **40 componentes** por mensagem e **4000 caracteres** de texto no total

> 💡 Ao usar `IsComponentsV2`, você **não pode** incluir `content`, `embeds` ou `stickers` na mensagem. Use `TextDisplayBuilder` no lugar do content.

---

## Tipagem de customId

Uma boa prática é padronizar os customId. Isso evita erro e deixa o código mais previsível.

```ts
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

## Criando o client

```ts
import { Client, GatewayIntentBits } from "discord.js"

export function createClient() {
  return new Client({
    intents: [GatewayIntentBits.Guilds]
  })
}
```

---

## Arquivo principal

```ts
import "dotenv/config"
import { createClient } from "./client/createClient"

const client = createClient()

client.once("clientReady", () => {
  console.log(`Bot online ${client.user?.tag}`)
})

client.login(process.env.DISCORD_TOKEN)
```

---

## Criando um painel com Container, botão e select

Aqui está um exemplo usando o `ContainerBuilder` como layout raiz, com seções, separador, botões e menu de seleção dentro dele.

```ts
import {
  ContainerBuilder,
  TextDisplayBuilder,
  SeparatorBuilder,
  SeparatorSpacingSize,
  SectionBuilder,
  ActionRowBuilder,
  ButtonBuilder,
  ButtonStyle,
  StringSelectMenuBuilder,
  MessageFlags
} from "discord.js"
import { encodeCustomId } from "./utils/customId"

export function buildPanel() {

  const header = new TextDisplayBuilder()
    .setContent("# Painel de Gerenciamento\nEscolha um módulo abaixo.")

  const separator = new SeparatorBuilder()
    .setDivider(true)
    .setSpacing(SeparatorSpacingSize.Small)

  const ticketsSection = new SectionBuilder()
    .addTextDisplayComponents(
      new TextDisplayBuilder().setContent("**Tickets**"),
      new TextDisplayBuilder().setContent("Abra e gerencie tickets de suporte.")
    )
    .setButtonAccessory(
      new ButtonBuilder()
        .setCustomId(encodeCustomId({ action: "tickets" }))
        .setLabel("Acessar")
        .setStyle(ButtonStyle.Primary)
    )

  const select = new StringSelectMenuBuilder()
    .setCustomId(encodeCustomId({ action: "menu" }))
    .setPlaceholder("Escolha um módulo")
    .addOptions(
      { label: "Tickets", value: "tickets" },
      { label: "Logs", value: "logs" },
      { label: "Moderação", value: "mod" }
    )

  const rowMenu = new ActionRowBuilder<StringSelectMenuBuilder>()
    .addComponents(select)

  const btnConfig = new ButtonBuilder()
    .setCustomId(encodeCustomId({ action: "config" }))
    .setLabel("Configurações")
    .setStyle(ButtonStyle.Secondary)

  const rowButtons = new ActionRowBuilder<ButtonBuilder>()
    .addComponents(btnConfig)

  const container = new ContainerBuilder()
    .setAccentColor(0x5865f2)
    .addTextDisplayComponents(header)
    .addSeparatorComponents(separator)
    .addSectionComponents(ticketsSection)
    .addActionRowComponents(rowMenu)
    .addActionRowComponents(rowButtons)

  return {
    components: [container],
    flags: MessageFlags.IsComponentsV2
  }
}
```

---

## Handlers separados

Separar os handlers evita deixar o evento `interactionCreate` gigante.

**`handlers/buttons.ts`**
```ts
import { ButtonInteraction } from "discord.js"
import { decodeCustomId } from "../utils/customId"

export async function handleButton(interaction: ButtonInteraction) {
  const data = decodeCustomId(interaction.customId)
  if (!data) return

  if (data.action === "config") {
    await interaction.reply({ content: "Abrindo configuração", flags: 64 })
  }

  if (data.action === "tickets") {
    await interaction.reply({ content: "Sistema de tickets", flags: 64 })
  }
}
```

**`handlers/selectMenus.ts`**
```ts
import { StringSelectMenuInteraction } from "discord.js"

export async function handleSelectMenu(interaction: StringSelectMenuInteraction) {
  const value = interaction.values[0]

  const respostas: Record<string, string> = {
    tickets: "Sistema de tickets",
    logs: "Sistema de logs",
    mod: "Sistema de moderação"
  }

  await interaction.reply({ content: respostas[value] ?? "Módulo não encontrado" })
}
```

---

## interactionCreate

```ts
import { Client, Events } from "discord.js"
import { handleButton } from "../handlers/buttons"
import { handleSelectMenu } from "../handlers/selectMenus"

export function registerInteractionCreate(client: Client) {
  client.on(Events.InteractionCreate, async (interaction) => {
    if (interaction.isButton()) {
      await handleButton(interaction)
    }

    if (interaction.isStringSelectMenu()) {
      await handleSelectMenu(interaction)
    }
  })
}
```

---

## Enviando o painel

```ts
const panel = buildPanel()

interaction.reply(panel)
```

---

## Algumas boas práticas

- Separe handlers sempre que possível
- Evite colocar lógica grande dentro de eventos
- Padronize customId com encode/decode
- Use TypeScript para evitar erros comuns
- Respeite os limites: máximo de **40 componentes** por mensagem e **4000 caracteres** de texto no total
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
