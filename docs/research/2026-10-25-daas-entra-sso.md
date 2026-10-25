# Recherche: Entra-SSO ohne FAS auf Citrix DaaS übertragen

| Feld | Wert |
|---|---|
| Status | abgeschlossen, offene Fragen unten |
| Datum | 2026-10-25 |
| Issue | [#3](https://github.com/dimi4ik/citrix-entra-sso/issues/3) |
| Tiefe | gründlich, Varianten A und C vertieft |
| Quellen | 22, Abruf aller Quellen am 2026-10-25 (Liste am Ende) |
| Umfang | 1 430 Wörter in Abschnitt 1 bis 5 (`wc -w`) |
| Sprache | Deutsch, als Ausnahme von der Englisch-Regel in `CLAUDE.md` (entschieden von Dima, 2026-10-25) |

Begriffe: **VDA** = die virtuelle Maschine, auf der der Desktop läuft. **Join-Art** = wie die Maschine einem
Verzeichnis angehört (nur AD, Entra hybrid joined = AD plus in Entra registriert, Entra joined = nur Entra).
**Client-App** = die Entra-App, über die der Browser das Token für den VDA holt. **[Q1]** usw. verweisen auf die
Quellenliste.

## 1. Fragestellung

Dieses Repo baut Entra-SSO ohne FAS für on-prem CVAD 2607 LTSR: NetScaler Gateway (OIDC) → StoreFront → Broker-Site
mit den site-weiten `AzureSSOn*`-Werten → Entra-hybrid-joined VDAs. Zu klären war, wie sich das auf Citrix DaaS
überträgt, und zwar für diese Umgebung (Angaben von Dima, 2026-10-25):

| Merkmal | Wert |
|---|---|
| Edition | DaaS Premium |
| Anmeldung an Workspace (IdP in Citrix Cloud) | Microsoft Entra ID |
| Zugang | Citrix Workspace mit Gateway Service |
| VDAs | heute nur AD-joined, in Azure, Windows Server 2025 / Windows 11 24H2, VDA 2507 oder neuer |
| Wechsel der Join-Art | denkbar |
| mTLS (Client-Zertifikat vor der Anmeldung) | offen |

Nicht Gegenstand: on-prem CVAD, FAS als Zielbild, Code.

## 2. Optionen

### A: Cloud-Weg mit hybrid joined VDAs (vertieft)

**Vorgehen.** Workspace bleibt mit Entra als IdP, Zugang bleibt Gateway Service. Die VDAs werden Entra hybrid joined:
neuer MCS-Katalog mit der Maschinenidentität „Microsoft Entra hybrid joined“, OU in Entra Connect Sync oder Cloud Sync
synchronisiert [Q6][Q14][Q15]. In Entra: die beiden Citrix-Apps per Admin-Consent anlegen (Resource-App
`3a510bb1-…`, Client-App Citrix-Workspace `85651ebe-…`), an der Resource-App das RDS-Protokoll einschalten und die
Client-App als „approved client app“ eintragen, optional eine Gerätegruppe [Q1][Q2]. Für hybrid zusätzlich ein
Entra-Kerberos-Serverobjekt in der AD-Domäne [Q1]. In Citrix: Delivery Group auf `MachineLogOnType HybridAzureAd`,
dann den Workspace-Schalter `AzureAdSsoEnabled` [Q1][Q4].

Übersicht: [architecture-overview-daas.svg](../img/architecture-overview-daas.svg) (Plan, nicht im Lab geprüft).

**Vorteile.** Kein NetScaler, kein StoreFront, keine eigene Client-App: „Citrix DaaS uses a Citrix managed Azure
client to handle the SSO“ [Q3]. Die VDAs behalten AD (Gruppenrichtlinien, Dateifreigaben). Microsoft führt hybrid
joined und nicht-persistent mit Managed-Anmeldung als „Limited“ und nennt Citrix Cloud ausdrücklich als
unterstützten Anbieter [Q13].

**Nachteile.** Ein bestehender Katalog mit AD-Identität lässt sich laut Community nicht per PowerShell umstellen, es
braucht einen neuen Katalog [Q19]; die Citrix-Seite sagt dazu nichts [Q6]. Der hybrid join beim ersten Start kann
bis 30 Minuten dauern [Q6]. Privilegierte Gruppen (z. B. Domain Admins) bekommen ohne RODC-Freigabe kein SSO [Q1].

**Aufwand.** Mittel: Sync-Konfiguration, Kerberos-Serverobjekt, neuer Katalog, Entra-Teil, zwei Citrix-Schalter.

### C: Entra joined VDAs (vertieft)

**Vorgehen.** Neuer MCS-Katalog mit Identität „Microsoft Entra joined“, nur in Azure, über „Machine Profile“,
Rendezvous V2 an [Q7]. Entra-Teil wie bei A, Delivery Group auf `MachineLogOnType AzureAd` [Q1].

**Vorteile.** Kein AD-Sync der Geräte, kein Kerberos-Serverobjekt, kein Warten auf den Join-Sync. Citrix listet
Entra joined mit Gateway Service und Workspace als unterstützt [Q1]; ein Leserbericht bestätigt den Betrieb in DaaS
[Q20]. Die VDAs laufen bereits in Azure, die
Bedingung „Azure only“ [Q7] ist erfüllt.

**Nachteile.** Kein Zugriff auf AD-Ressourcen ohne Zusatzaufwand (Annahme, in den gelesenen Quellen nicht
behandelt). Microsoft führt Entra joined + nicht-persistent als „No“ und Entra joined + persistent als „Limited“, nur
für AVD, Windows 365 und Amazon WorkSpaces [Q13]. Citrix dagegen nennt „Persistent and Non-persistent“ [Q7]. Die
Quellen widersprechen sich. Ohne SSO zeigt der erste Start bei nicht-persistenten Maschinen den zuletzt angemeldeten
Benutzer [Q7].

**Aufwand.** Mittel bis hoch: neuer Katalog, Apps und Richtlinien ohne AD neu denken (Intune statt GPO).

### B: eigener NetScaler und StoreFront in der Resource Location (nicht vertieft)

Nah am heutigen Repo und der einzige belegte Weg zu mTLS. StoreFront on-prem gegen DaaS braucht zusätzlich einen
Bypass am Gateway für `/Citrix/<StoreWeb>/Tickets/RedeemStoreTicket` per Content Switching und nutzt ebenfalls die
Citrix-Client-App [Q3]. Nur für hybrid joined, nicht für Entra joined [Q1]. Nicht vertieft, weil Dima Gateway
Service nutzt; Wiedervorlage, falls mTLS Pflicht wird.

### D: FAS für die AD-only VDAs (verworfen)

AD-only VDAs bekommen kein Entra-SSO; Citrix verweist dort auf FAS [Q1][Q3][Q5]. FAS und Entra-SSO laufen
nebeneinander, getrennt je Delivery Group [Q5][Q19]. Verworfen als Zielbild, weil das Repo gerade „ohne FAS“ ist.
Als Übergang für Kataloge, die noch nicht umgestellt sind, bleibt es nutzbar.

## 3. Erkenntnisse

**Unterstützung.** DaaS kann Entra-SSO ohne FAS („Microsoft Entra single sign-on“) für „Microsoft Entra joined or
Microsoft Entra hybrid joined session hosts“ [Q1]. AD-only ist nicht dabei.

**Versionen** [Q1]:

| Teil | Mindestens |
|---|---|
| VDA | 2507 |
| Windows 11 24H2 | Build 26100.6725 (KB5065789) |
| Windows Server 2025 | Build 26100.32230 (KB5073379) |
| Workspace app Windows | 2507 |
| Workspace app Mac | 2508, hybrid 2511 |
| Workspace app Linux | 2508, hybrid 2601 |
| HTML5, iOS, Android, ChromeOS | 2511 |
| Hybrid-Start (Browser → Workspace app) | Edge oder Chrome mit Citrix Web Extension |

Dimas VDAs erfüllen das (Server 2025 / Windows 11 24H2, VDA ≥ 2507); die Build-Nummern sind noch zu prüfen.

**IdP in Workspace.** Nur Entra ID oder SAML; AD, AD + Token, Okta, NetScaler Gateway und Adaptive Authentication
sind „No“ [Q1]. Dimas Einstellung (Entra) passt.

**Wo eingestellt wird – der große Unterschied zu on-prem.**

| Stelle | on-prem (dieses Repo) | DaaS |
|---|---|---|
| Client-App | eigene Single-Tenant-App | Citrix-Workspace `85651ebe-…`, von Citrix verwaltet [Q1][Q3] |
| SSO-Werte am Broker | `Set-BrokerSite -AzureSSOn*`, site-weit | im DaaS-SDK nicht dokumentiert [Q8], in der CVAD-SDK schon [Q9] |
| Schalter Store | `Set-STFStoreLaunchOptions -EntraIdSsoEnabled` [Q3] | `Set-StoreConfigurations … -AzureAdSsoEnabled $True`, Modul 1.0.6 [Q1][Q4]; KB nennt `Set-WorkspaceCustomization` [Q5] |
| Reichweite | eine Store-Herkunft je Site | „a global setting that will be applied to all the stores in your Citrix Cloud tenant“ [Q1] |
| Delivery Group | `MachineLogOnType HybridAzureAd` | `HybridAzureAd` oder `AzureAd` [Q1] |
| Gateway | eigener NetScaler, OIDC `oid`, Token-Rewrite | Gateway Service, nichts zu konfigurieren (Annahme: keine Gateway-Einstellung in [Q1] genannt) |

Die site-weite Grenze aus `README.md` wird in DaaS zu einer mandantenweiten: ein Schalter für alle Stores des
Citrix-Cloud-Mandanten. Ob es dahinter `AzureSSOn*`-Werte gibt, die Citrix selbst setzt, ist nicht belegt. Eigene
Messung: die DaaS-SDK-Seite zu `Set-BrokerSite` enthält andere Parameter (`-TrustRequestsSentToTheXmlServicePort`),
aber keinen `AzureSSOn*`; die CVAD-Seite derselben Cmdlet enthält alle vier [Q8][Q9].

**Entra-Seite.** Das Protokoll ist dasselbe wie on-prem: `remoteDesktopSecurityConfiguration` am Service Principal
der Resource-App, `approvedClientApps`, `targetDeviceGroups` (höchstens 10; ohne Gruppe merkt sich Entra „up to 15
hosts for 30 days“) [Q10][Q12]. Nur die eingetragene Client-App ist eine andere: Citrix-Workspace statt der eigenen
App [Q2]. Rechte: Application Administrator oder Cloud Application Administrator, kleinster Graph-Scope
`Application-RemoteDesktopConfig.ReadWrite.All` [Q11].

**Conditional Access.** Nur auf die Resource-App, nicht auf die Client-App: „must be applied to the Citrix Resource
application, not the Citrix Client application or Microsoft applications“ [Q2], ebenso [Q21]. Für das gleiche
Protokoll bei AVD rät Microsoft, die Anmeldehäufigkeit der SSO-App gleich oder länger als die der Anmeldung zu
setzen [Q16]. Das deckt sich mit dem Standard
`applications: ["resource"]` in `New-EntraSsoConditionalAccess.ps1`; die Option `client` würde in DaaS auf eine
Citrix-App zielen und widerspricht der Doku.

**Bekannte Fehlerbilder** [Q1][Q17]: Auto Client Reconnect ist mit SSO aus; die Richtlinie „Auto client reconnect
authentication“ auf Pflicht bricht SSO; die Intune-Security-Baseline schaltet „Allow Custom SSPs and APs to be loaded
into LSASS“ ab und blockiert SSO; AADSTS293005 → Workspace app neu starten; 30 s Wartezeit, wenn Client oder Web
Extension kein SSO können oder der Schalter vor der Azure-Konfiguration gesetzt wurde; ohne „Angemeldet bleiben“
(KMSI) scheitert SSO auf Mac und Linux; Entra External ID (Gäste) nicht unterstützt. Das 30-s-Muster entspricht der
on-prem-Signatur in `docs/troubleshooting.md` (CloudAP `0xC0048422`).

**mTLS.** Kein Beleg für Client-Zertifikate am Gateway Service, weder dafür noch dagegen. Adaptive Authentication
kann Zertifikate, ist aber als IdP für Entra-SSO „No“ [Q1]. Device Posture kann Gerätezertifikate prüfen [Q22]; ob
das mit Entra-SSO zusammen läuft, ist offen.

## 4. Risiken

| Variante | Risiko |
|---|---|
| A | Neuer Katalog nötig, Umzug der Benutzer; Join-Verzögerung beim Bereitstellen; veraltete Geräteobjekte bei nicht-persistenten Maschinen (Microsoft: löschen ab 15 Tagen ohne Anmeldung) [Q13] |
| A | Kerberos-Serverobjekt und RODC-Freigaben sind neue Teile im AD, mit Rechten auf Domänenebene [Q1][Q18] |
| C | Microsoft-Support für Entra joined mit Citrix nicht in der Microsoft-Tabelle [Q13]; Apps mit AD-Abhängigkeit laufen nicht ohne Weiteres |
| A, C | Schalter `AzureAdSsoEnabled` gilt für alle Stores des Mandanten; vor der Azure-Konfiguration gesetzt → 30-s-Starts für alle [Q1] |
| A, C | Kein mTLS-Weg belegt; wer mTLS braucht, landet bei B |
| A, C | Verhalten der Anmeldehäufigkeit (Sign-in frequency) in DaaS ungetestet; on-prem scheiterte der erste Start nach Ablauf (`docs/troubleshooting.md`) |
| B | Doppelter Betrieb NetScaler + StoreFront für einen Cloud-Dienst; Ticket-Bypass ist eine zusätzliche, ungeschützte Route [Q3] |

## 5. Empfehlung

**Variante A**: Workspace mit Entra, Gateway Service, Citrix-Client-App, neuer MCS-Katalog mit hybrid joined VDAs.
Sie hält AD für die Anwendungen, braucht keine eigene Infrastruktur und ist der einzige Weg, für den Citrix und
Microsoft sich beim nicht-persistenten Betrieb nicht widersprechen [Q1][Q6][Q13]. Variante C bleibt eine Option für
persistente, AD-freie Arbeitsplätze; mTLS wird als eigene Frage geklärt, bevor B ins Spiel kommt.

Als Nächstes zu klären (lesend, umkehrbar, vor jeder Frage an Citrix): ob `Get-BrokerSite` im eigenen DaaS-Mandanten
`AzureSSOn*`-Eigenschaften zeigt. Vorhersage: die Eigenschaften fehlen oder sind leer.

## Offene Fragen

| # | Frage | Warum offen |
|---|---|---|
| O1 | Gibt es `AzureSSOn*` in DaaS, und setzt Citrix sie selbst? | DaaS-SDK-Doku nennt sie nicht [Q8]; kein Test im Mandanten |
| O2 | Welcher Befehl schaltet den Store: `Set-StoreConfigurations` [Q1][Q4] oder `Set-WorkspaceCustomization` [Q5]? Wie meldet sich das Modul an (`-ClientId`/`-ClientSecret`)? | Quellen widersprechen sich |
| O3 | Geht mTLS am Gateway Service? Geht Device Posture zusammen mit Entra-SSO? | keine Quelle gefunden |
| O4 | Wie verhält sich die Anmeldehäufigkeit an der Resource-App beim ersten Start in DaaS? | nur on-prem getestet |
| O5 | Katalogidentität AD → hybrid umstellbar oder Neubau? | nur Community [Q19], Citrix schweigt [Q6] |
| O6 | Sind Dimas Kataloge persistent oder nicht-persistent? | entscheidet zwischen A und C |
| O7 | Cloud Sync Device Sync: Vorschau oder allgemein verfügbar? | Seite sagt es nicht [Q15] |
| O8 | Welche Rolle braucht der Admin-Consent der Citrix-Apps? | nur Suchtreffer zu CTX665875, Seite nicht gelesen |
| O9 | Microsoft-Support für Entra joined mit Citrix (C) | Citrix sagt ja [Q7], Microsoft-Tabelle nennt Citrix nicht [Q13] |

## Quellen

Alle abgerufen am 2026-10-25. „Selbst geprüft“ = in dieser Sitzung direkt an der Seite nachgelesen; sonst über einen
Rechercheagenten, Wortlaut vor Einbau in Doku nochmals prüfen. [H] Hersteller, [C] Community.

| # | Quelle | Art | Geprüft |
|---|---|---|---|
| Q1 | [Citrix DaaS: Microsoft Entra single sign-on](https://docs.citrix.com/us/en/citrix-daas/install-configure/session-authentication/entra-sso.html) (veröffentlicht 2026-10-12) | H | selbst |
| Q2 | [Citrix DaaS: Entra SSO, Azure-Konfiguration](https://docs.citrix.com/en-us/citrix-daas/install-configure/session-authentication/entra-sso/entra-sso-azure-config.html) | H | Agent |
| Q3 | [StoreFront: Entra ID SSO](https://docs.citrix.com/us/en/storefront/current-release/stores/settings/entraid-sso.html) (2026-10-16) | H | selbst |
| Q4 | [Citrix Workspace: Authentication](https://docs.citrix.com/us/en/citrix-workspace/authentication.html) (2026-10-15) | H | selbst |
| Q5 | [CTX696865: Coexistence of FAS and non-FAS](https://support.citrix.com/external/article/CTX696865/citrix-coexistence-of-fas-and-nonfas-en.html) (ohne Datum) | H | selbst |
| Q6 | [Citrix DaaS: Hybrid joined machine identity](https://docs.citrix.com/en-us/citrix-daas/install-configure/machine-identities/hybrid-azure-active-directory-joined.html) (2026-10-12) | H | selbst |
| Q7 | [Citrix DaaS: Entra joined machine identity](https://docs.citrix.com/en-us/citrix-daas/install-configure/machine-identities/azure-active-directory-joined.html) (2026-10-12) | H | selbst |
| Q8 | [DaaS SDK: Set-BrokerSite](https://developer-docs.citrix.com/en-us/citrix-daas-sdk/broker/set-brokersite) | H | selbst (`curl` + `grep`: kein `AzureSSOn`) |
| Q9 | [CVAD SDK current release: Set-BrokerSite](https://developer-docs.citrix.com/en-us/citrix-virtual-apps-desktops-sdk/current-release/broker/set-brokersite) | H | selbst (`curl` + `grep`: alle vier `AzureSSOn*`) |
| Q10 | [Graph: remoteDesktopSecurityConfiguration](https://learn.microsoft.com/en-us/graph/api/resources/remotedesktopsecurityconfiguration?view=graph-rest-1.0) | H | Agent |
| Q11 | [Graph: remoteDesktopSecurityConfiguration update](https://learn.microsoft.com/en-us/graph/api/remotedesktopsecurityconfiguration-update?view=graph-rest-1.0) | H | Agent |
| Q12 | [AVD: Configure single sign-on](https://learn.microsoft.com/en-us/azure/virtual-desktop/configure-single-sign-on) | H | Agent |
| Q13 | [Entra: Device identity and desktop virtualization](https://learn.microsoft.com/en-us/entra/identity/devices/howto-device-identity-virtual-desktop-infrastructure) (ms.date 2026-06-04) | H | selbst |
| Q14 | [Entra: Configure hybrid join](https://learn.microsoft.com/en-us/entra/identity/devices/how-to-hybrid-join) | H | Agent |
| Q15 | [Entra Cloud Sync: device sync](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/device-sync) (2026-10-21) | H | Agent |
| Q16 | [AVD: Set up MFA / Conditional Access](https://learn.microsoft.com/en-us/azure/virtual-desktop/set-up-mfa) | H | Agent |
| Q17 | [CTX696795: DaaS Microsoft Entra single sign-on](https://support.citrix.com/external/article/CTX696795/citrix-daas-microsoft-entra-single-sign.html) | H | Agent |
| Q18 | [Ferroque: Citrix SSO Entra ID lessons learned](https://www.ferroquesystems.com/resource/citrix-sso-entra-id-lessons-learned/) | C | Agent |
| Q19 | [Julian Jakob: DaaS Entra ID SSO in co-existence with FAS](https://www.julianjakob.com/citrix-daas-entra-id-sso-in-co-existence-with-fas/) | C | Agent |
| Q20 | [Julian Jakob: DaaS Entra ID SSO with PRT and without FAS](https://www.julianjakob.com/citrix-daas-entra-id-sso-with-prt-and-without-fas/) | C | Agent |
| Q21 | [archy.net: Citrix native Entra SSO without FAS](https://www.archy.net/citrix-native-entra-sso-without-fas/) | C | Agent |
| Q22 | [Citrix Device Posture](https://docs.citrix.com/en-us/device-posture.html) | H | Agent |

Nicht lesbar: Reddit und community.citrix.com (Abruf gesperrt bzw. 403); Forenaussagen sind daher nicht eingeflossen.
