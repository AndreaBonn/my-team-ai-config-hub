> **English:** [Read in English](./readme-tech-security.md)
>
> **Docs:** [README](./README.it.md) | [Architettura](./readme-tech-architecture-it.md)

# Note di Audit Sicurezza

## npm audit - Vulnerabilita transitive (2026-05-17)

`npm audit` riporta 12 vulnerabilita (8 low, 4 moderate). Tutte sono in dipendenze transitive di `next` e `firebase-admin`, senza percorsi di exploit diretti nel contesto di questa applicazione.

### Riepilogo

| Package | Severita | Origine | Sfruttabile qui? | Motivazione |
|---------|----------|---------|-------------------|-------------|
| postcss (via next) | Moderata | XSS nell'output di CSS stringify | No | Non processiamo CSS non fidato a runtime; PostCSS gira solo in fase di build |
| @tootallnate/once (via firebase-admin) | Bassa | Scoping errato del control flow | No | Dipendenza transitiva di http-proxy-agent, non invocata nei nostri code path |
| Altre 10 | Bassa | Varie | No | Tutte in code path di build-time o non utilizzati nelle dipendenze transitive |

### Decisione

- Non risolvibili senza bump di versione major di `next` o `firebase-admin`, che introdurrebbero breaking change.
- `npm audit fix --force` farebbe il downgrade di `next` alla 9.x, non accettabile.
- La CI esegue `npm audit --audit-level=high` per intercettare vulnerabilita high/critical.
- Accettiamo i rischi transitivi moderate/low e monitoriamo i fix upstream.

### Rivalutazione

Revisione mensile o al bump di `next` / `firebase-admin` a una nuova versione major.
