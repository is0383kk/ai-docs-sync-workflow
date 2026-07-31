---
read_when:
    - Je hebt gestructureerde bestandsbewerkingen in meerdere bestanden nodig
    - Je wilt op patches gebaseerde bewerkingen documenteren of fouten erin opsporen
summary: Pas patches op meerdere bestanden toe met de tool apply_patch
title: apply_patch-tool
x-i18n:
    generated_at: "2026-07-27T05:35:43Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 1c0422550ea8d9b0cb6b0ea22d7dcaecc462426f9600003f70c177746f30a3d9
    source_path: tools/apply-patch.md
    workflow: 16
---

Pas bestandswijzigingen toe met een gestructureerde patchindeling. Dit is ideaal voor bewerkingen in meerdere bestanden
of met meerdere hunks, waarbij één aanroep van `edit` kwetsbaar zou zijn.

De tool accepteert één tekenreeks `input` die een of meer bestandsbewerkingen omvat:

```text
*** Begin Patch
*** Add File: path/to/file.txt
+regel 1
+regel 2
*** Update File: src/app.ts
@@ optionele wijzigingscontext
-oude regel
+nieuwe regel
*** Delete File: obsolete.txt
*** End Patch
```

## Parameters

- `input` (vereist): Volledige patchinhoud, inclusief `*** Begin Patch` en `*** End Patch`.

## Opmerkingen

- Patchpaden ondersteunen relatieve paden (vanaf de werkruimtemap) en absolute paden.
- `tools.exec.applyPatch.workspaceOnly` is standaard `true` (binnen de werkruimte). Stel dit alleen in op `false` als je bewust wilt dat `apply_patch` buiten de werkruimtemap schrijft of verwijdert.
- Gebruik `*** Move to:` binnen een hunk van `*** Update File:` om bestanden te hernoemen.
- `*** End of File` markeert indien nodig een invoeging uitsluitend aan het einde van het bestand.
- Standaard ingeschakeld voor elk model. Stel `tools.exec.applyPatch.enabled: false`
  in om dit uit te schakelen, of beperk het tot specifieke modellen met
  `tools.exec.applyPatch.allowModels` (accepteert onbewerkte id's zoals `gpt-5.4` of volledige
  id's zoals `openai/gpt-5.4`).
- De configuratie staat onder `tools.exec.applyPatch.*`.

## Voorbeeld

```json
{
  "tool": "apply_patch",
  "input": "*** Begin Patch\n*** Update File: src/index.ts\n@@\n-const foo = 1\n+const foo = 2\n*** End Patch"
}
```

## Gerelateerd

<CardGroup cols={2}>
  <Card title="Diffs" href="/nl/tools/diffs" icon="code-compare">
    Alleen-lezen diffweergave voor het presenteren van wijzigingen.
  </Card>
  <Card title="Uitvoeringstool" href="/nl/tools/exec" icon="terminal">
    Uitvoering van shellopdrachten vanuit de agent.
  </Card>
  <Card title="Code-uitvoering" href="/nl/tools/code-execution" icon="square-code">
    Gesandboxte externe Python-analyse met xAI.
  </Card>
</CardGroup>
