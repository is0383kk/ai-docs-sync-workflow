---
read_when:
    - Beanspruchen einer Organisation, Marke, eines Paketbereichs, Eigentümer-Handles, Skill-Slugs oder Paket-Namensraums
    - Auflösen eines bereits beanspruchten oder reservierten Namespace
    - Entscheidung zwischen Meldung, Einspruch und Namespace-Anspruch
sidebarTitle: Org and Namespace Claims
summary: So beantragen Sie eine ClawHub-Prüfung bei Streitfällen über die Inhaberschaft von Organisationen, Marken, Inhaber-Handles, Paket-Scopes, Skill-Slugs oder Namespaces.
title: Organisations- und Namespace-Ansprüche
x-i18n:
    generated_at: "2026-07-26T18:16:09Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 77a4d8090b55298c401154d116d93d4f8139d40983a45982288d8e48bcea40fb
    source_path: clawhub/namespace-claims.md
    workflow: 16
---

# Ansprüche auf Organisationen und Namespaces

ClawHub verwendet Inhaber-Handles, Organisations-Handles, Skill-Slugs, Plugin-Paketnamen und
Paket-Scopes als öffentliche Namespaces. Wenn ein Namespace offenbar zu einem
realen Projekt, einer Marke, einem Paket-Ökosystem oder einer Organisation gehört, auf
ClawHub jedoch bereits beansprucht, reserviert, irreführend oder umstritten ist, bitten Sie das Personal um eine Überprüfung
mithilfe des
[Formulars für Ansprüche auf Organisationen/Namespaces](https://github.com/openclaw/clawhub/issues/new?template=org-namespace-claim.yml).

Verwenden Sie diesen Weg für öffentliche, nicht vertrauliche Überprüfungen der Inhaberschaft. Verwenden Sie für Namespace-Ansprüche weder produktinterne
Meldungen noch das Einspruchsformular für Konten.

## Wann Sie einen Anspruch einreichen sollten

Reichen Sie einen Namespace-Anspruch ein, wenn ClawHub-Mitarbeiter Ihrer Ansicht nach prüfen sollten, ob ein
Namespace aufgrund realer Inhaberschaft reserviert, übertragen, umbenannt, ausgeblendet, unter Quarantäne gestellt, mit einem Alias versehen
oder anderweitig geändert werden sollte.

Beispiele:

- ein Organisations-Handle, das Ihrer GitHub-Organisation, Ihrem Projekt, Ihrem Unternehmen oder Ihrer Community entspricht
- ein Paket-Scope wie `@example-org/*`, unter dem ausschließlich der
  entsprechende ClawHub-Inhaber veröffentlichen sollte
- ein Skill-Slug oder Plugin-Paketname, der offenbar die Identität eines Projekts vortäuscht
- eine Streitigkeit über eine Marke, ein Warenzeichen, die Umbenennung eines Projekts oder den Verlauf eines Pakets
- ein gelöschter, inaktiver oder nicht erreichbarer Inhaber, der den rechtmäßigen Namespace-Inhaber
  blockiert

Wenn der Eintrag über die Inhaberschaftsstreitigkeit hinaus unsicher, bösartig oder irreführend ist,
befolgen Sie außerdem die entsprechende Moderations- oder Sicherheitsanleitung. Das Formular für Namespace-Ansprüche
dient der Überprüfung der Inhaberschaft, nicht der dringenden Offenlegung von Sicherheitslücken.

## Vor dem Einreichen

Vergewissern Sie sich zunächst, dass Sie mit dem Inhaber veröffentlichen, der dem Namespace entspricht.
Bei Plugin-Paketen müssen Scoped-Namen wie `@example-org/example-plugin`
unter dem entsprechenden Inhaber `example-org` veröffentlicht werden.

Wenn Sie den aktuellen Inhaber verwalten können, korrigieren Sie den Namespace direkt, indem Sie die betroffene Ressource veröffentlichen,
umbenennen, übertragen, ausblenden oder löschen. Reichen Sie einen Anspruch ein,
wenn Sie den aktuellen Inhaber nicht verwalten können oder Mitarbeiter eine
Streitigkeit klären müssen.

## Beizufügende Nachweise

Verwenden Sie öffentliche, nicht vertrauliche Nachweise. Hilfreiche Belege sind unter anderem:

- Verlauf der GitHub-Organisation, des Repositorys, der Releases oder der Maintainer
- offizielle Projektdokumentation, die den Namespace nennt
- Nachweis über eine Domain oder offizielle E-Mail-Domain
- Kontrolle über den Scope bei npm, PyPI, crates.io oder einer anderen Paket-Registry
- Nachweise über Warenzeichen, Marken- oder Projekteigentum, die öffentlich
  besprochen werden können
- Verlauf des Quell-Repositorys, Paketverlauf oder öffentliche Hinweise zur Umbenennung
- Links zum umstrittenen ClawHub-Inhaber, Skill, Plugin, Paket oder Issue

Erläutern Sie, was jeder Link belegt. Mitarbeiter sollten die
Beziehung nachvollziehen können, ohne private Anmeldedaten oder Geheimnisse zu benötigen.

## Was Sie nicht angeben sollten

Veröffentlichen Sie keine Geheimnisse oder privaten Nachweise in einem öffentlichen GitHub-Issue. Geben Sie Folgendes nicht an:

- API-Token, Signaturschlüssel oder Anmeldedaten
- DNS-Challenge-Token
- private Rechtsunterlagen oder Verträge
- persönliche Identitätsdokumente
- private E-Mails, private Sicherheitsberichte oder vertrauliche Kundendaten

Im Anspruchsformular wird gefragt, ob vertrauliche Nachweise einen privaten Kanal zum Personal erfordern.
Verwenden Sie diese Option, statt vertrauliches Material öffentlich zu veröffentlichen.

## Mögliche Ergebnisse

Je nach Nachweisen und Risiko können ClawHub-Mitarbeiter einen Namespace reservieren,
die Inhaberschaft übertragen, eine Ressource umbenennen, einen bestehenden Eintrag ausblenden oder unter Quarantäne stellen,
einen Alias oder eine Weiterleitung hinzufügen, weitere Nachweise anfordern oder die Anfrage ablehnen.

Die Überprüfung eines Namespace garantiert nicht, dass jeder übereinstimmende Name übertragen wird.
Das Personal wägt öffentliche Nachweise, bestehende Nutzung, Sicherheitsrisiken und Auswirkungen auf Benutzer ab.

## Verwandte Dokumentation

- [Veröffentlichen](/de/clawhub/publishing)
- [Fehlerbehebung](/de/clawhub/troubleshooting#publish-fails-because-a-namespace-is-claimed-or-reserved)
- [Moderation und Kontosicherheit](/de/clawhub/moderation)
- [Sicherheit](/de/clawhub/security)
