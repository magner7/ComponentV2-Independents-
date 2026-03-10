# 🫧 Components V2 com TypeScript

Um guia objetivo sobre como utilizar Components V2 no Discord com TypeScript. A proposta é apresentar uma base limpa e bem estruturada, fácil de escalar em projetos maiores.

---

## Sobre Components V2

Os Components V2 permitem construir interfaces ricas dentro do Discord utilizando uma abordagem baseada inteiramente em componentes. Em vez de combinar `content` e `embeds`, o layout é montado com blocos como texto, seções, botões e menus — tudo dentro de um `ContainerBuilder`.

É especialmente útil para painéis interativos, dashboards e sistemas de configuração. Com TypeScript, a manutenção e a evolução do projeto se tornam muito mais previsíveis.

> ⚠️ Requer discord.js `14.19.3` ou superior. Verifique com `npm list discord.js`.

---

## Dependências

```bash
npm install discord.js@latest dotenv
npm install -D typescript ts-node @types/node
```

---

## Documentação

- [Discord API — Components](https://discord.com/developers/docs/components/reference)
- [discord.js — Display Components](https://discordjs.guide/legacy/popular-topics/display-components)
- [discord.js — Docs completa](https://discord.js.org/docs)
- [Mock visual de mensagens](https://discord.builders)

---

## Estrutura do projeto

Estrutura básica recomendada para manter o projeto organizado desde o início:

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

Separar builders de handlers evita arquivos extensos e facilita a manutenção.

---

## Como o layout funciona no Container

O `ContainerBuilder` é o componente raiz — todos os demais elementos são adicionados dentro dele para compor o layout da mensagem.

```
ContainerBuilder               ← componente raiz do layout
├── TextDisplayBuilder         ← texto com markdown (substitui o content)
├── SeparatorBuilder           ← divisória visual entre blocos
├── SectionBuilder             ← texto com botão ou thumbnail como acessório
│   ├── TextDisplayBuilder (até 3)
│   └── ButtonBuilder ou ThumbnailBuilder
├── ActionRowBuilder           ← linha de botões ou select menus
│   ├── ButtonBuilder
│   └── StringSelectMenuBuilder
└── MediaGalleryBuilder        ← grade de até 10 imagens
```

> ⚠️ Ao utilizar `IsComponentsV2`, os campos `content`, `embeds` e `stickers` não são permitidos na mensagem. Utilize `TextDisplayBuilder` no lugar do `content`.

---

## Tipagem de customId

Padronizar os `customId` evita erros e torna o código mais previsível conforme o bot cresce:

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
  console.log(`Bot online: ${client.user?.tag}`)
})

client.login(process.env.DISCORD_TOKEN)
```

---

## Criando um painel com Container, botão e select

Exemplo completo utilizando `ContainerBuilder` como layout raiz, com cabeçalho, seção com botão acessório, select menu e botão de ação:

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
    .setContent("# Painel de Gerenciamento\nSelecione um módulo abaixo.")

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
    .setPlaceholder("Selecione um módulo")
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

Centralizar toda a lógica no evento `interactionCreate` torna o código difícil de manter. O recomendado é separar cada tipo de interação em seu próprio arquivo:

**`handlers/buttons.ts`**
```ts
import { ButtonInteraction } from "discord.js"
import { decodeCustomId } from "../utils/customId"

export async function handleButton(interaction: ButtonInteraction) {
  const data = decodeCustomId(interaction.customId)
  if (!data) return

  if (data.action === "config") {
    await interaction.reply({ content: "Abrindo configurações.", flags: 64 })
  }

  if (data.action === "tickets") {
    await interaction.reply({ content: "Sistema de tickets.", flags: 64 })
  }
}
```

**`handlers/selectMenus.ts`**
```ts
import { StringSelectMenuInteraction } from "discord.js"

export async function handleSelectMenu(interaction: StringSelectMenuInteraction) {
  const value = interaction.values[0]

  const respostas: Record<string, string> = {
    tickets: "Módulo de tickets carregado.",
    logs: "Módulo de logs carregado.",
    mod: "Módulo de moderação carregado."
  }

  await interaction.reply({ content: respostas[value] ?? "Módulo não encontrado." })
}
```

---

## interactionCreate

Com os handlers separados, o evento concentra apenas o roteamento das interações:

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

## Boas práticas

- Separe handlers por tipo de interação para manter o evento enxuto
- Evite lógica complexa dentro de eventos, delegue para funções específicas
- Padronize os `customId` com encode/decode desde o início do projeto
- Utilize TypeScript para garantir tipagem e evitar erros em tempo de execução
- Respeite os limites do Discord: máximo de **40 componentes** por mensagem e **4000 caracteres** de texto no total
