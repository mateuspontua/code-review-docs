# 📊 Code Review Guide - Resumo Executivo

## Visão Rápida

Este documento é um **checklist coeso e honesto** para garantir que Pull Requests no projeto Pontua sigam os padrões arquitetônicos, SOLID, e boas práticas de React.

---

## 🚀 Início Rápido do Review

Ao revisar uma PR, verifique rapidamente:

### 1️⃣ Escopo (2 min)
- [ ] Uma mudança? Uma feature? Um bugfix?
- [ ] < 15 arquivos alterados?
- [ ] Descrição clara na PR?

### 2️⃣ Código (5 min)
- [ ] Nenhum texto hardcoded (sem `useTranslation`)?
- [ ] Magic numbers sem constantes?
- [ ] Muitas props ou prop-drilling?

### 3️⃣ Arquitetura (5 min)
- [ ] Domain/Service segue `DOMAIN_SERVICE_PATTERN.md`?
- [ ] Estrutura de pastas correta?
- [ ] Sem duplicação óbvia?

### 4️⃣ React (3 min)
- [ ] memo() usado corretamente?
- [ ] lazy() para componentes pesados?
- [ ] Deps array completo?

### 5️⃣ SOLID (2 min)
- [ ] SRP: cada coisa faz uma coisa?
- [ ] OCP: adicionar feature requer mudança no código existente?
- [ ] Sem muita responsabilidade em uma classe?

---

## ✅ Quando Aprovar

```
✅ Approvar quando:
  • Segue todos os padrões
  • Código limpo
  • Sem magic numbers
  • Tradução correta
  • Tamanho razoável
  • React best practices
```

---

## ⚠️ Quando Comentar

```
💬 Comentar quando:
  • Sugestão de melhoria (não crítico)
  • Performance optimization
  • Alternativa mais limpa
  • Documentação faltando
```

---

## ❌ Quando Rejeitar

```
🚫 Rejeitar quando:
  ✗ Viola DOMAIN_SERVICE_PATTERN.md
  ✗ Texto hardcoded sem useTranslation
  ✗ Magic numbers
  ✗ Prop-drilling excessivo (5+ props)
  ✗ PR muito grande (> 15 arquivos)
  ✗ Testes quebrados
  ✗ SOLID violation óbvia
```

---

## 📋 Checklist Rápido por Tipo

### 🆕 Nova Feature
- [ ] Escopo bem definido?
- [ ] Domain/Service estruturados?
- [ ] Testes inclusos?
- [ ] Documentação atualizada?

### 🐛 Bugfix
- [ ] Testa o bug?
- [ ] Não quebra outros testes?
- [ ] Causa está tratada?

### 🔧 Refactor
- [ ] Mesmo comportamento?
- [ ] Código mais legível?
- [ ] Testes passam?

### 📚 Documentação
- [ ] Clara e completa?
- [ ] Exemplos corretos?
- [ ] Links funcionam?

---

## 🎯 Métricas de Aprovação

| Critério | ✅ Ideal | ⚠️ Alerta | ❌ Rejeitar |
|----------|----------|----------|-----------|
| Arquivos | 1-3 | 4-8 | > 15 |
| Linhas | 100-300 | 300-500 | > 1000 |
| Tempo Review | < 10 min | 10-20 min | > 30 min |
| Props/Componente | < 5 | 5-8 | > 10 |
| Complexidade | Simples | Média | Alta |

---

## 💡 Dicas de Reviewer

1. **Seja Construtivo**
   - "Como poderíamos..." em vez de "Por que você..."
   - Ofereça alternativas, não apenas crítica

2. **Aprove Pequenas Coisas**
   - Use "Comment" para sugestões menores
   - Reserve "Request Changes" para problemas reais

3. **Reconheça Bom Código**
   - "Adorei essa abordagem com useCallback aqui!"
   - Mentalidade positiva melhora código futuro

4. **Seja Rápido**
   - Revisar em < 24 horas mantém momentum
   - Deixar PR pendurada é pior que pequenas críticas

5. **Siga o Padrão**
   - Use este guia como referência
   - Seja consistente

---

## 🔗 Referências Rápidas

### Arquitetura
- 📖 [DOMAIN_SERVICE_PATTERN.md](./DOMAIN_SERVICE_PATTERN.md) - Guia completo de padrão
- 📁 `app/domain/` - Exemplos de implementação
- 📁 `app/service/core/` - Serviços de API

### Tradução
- 🌍 `app/translate/translations.d.ts` - Tipos de tradução
- 🌍 `app/translate/locales/ts/pt.br.ts` - Traduções reais

### React
- 📚 [React Docs](https://react.dev) - Documentação oficial
- 🔍 [React DevTools Profiler](https://react.dev/learn/react-developer-tools) - Para analisar renders

### SOLID
- 📖 [Uncle Bob - SOLID](https://en.wikipedia.org/wiki/SOLID) - Princípios fundamentais

---

## 🎓 Exemplo de Review Completo

### ❌ Review Ruim
```
"Isso está errado. Refaz."
```

### ✅ Review Bom
```
**Sugestão:** Mover este código para um custom hook

**Contexto:** Segundo nossa arquitetura, lógica de estado 
deve estar em hooks, não em componentes.

**Código sugerido:**
```typescript
function useUserFiltering(users, filter) {
    return useMemo(() => users.filter(...), [users, filter]);
}
```

**Por quê:** Facilita reutilizar a lógica em outros 
componentes e torna testável isoladamente.
```

---

## 🚩 Red Flags Imediatas

Quando ver isso, para e solicita mudanças:

```
❌ "export { anything } from 'axios'" em componente
❌ "const LIMITE = 50;" sem ser uma constante de config
❌ <button>Salvar</button> - sem useTranslation
❌ {condition && <Modal />} - sem justificativa de por quê
❌ <Component {...props} /> - repassando tudo sem usar
❌ 47 arquivos alterados em uma PR
❌ if (type === 'a') { ... } if (type === 'b') { ... } if ...
```

---

## ✨ Green Lights

Quando ver isso, é bom sinal:

```
✅ Arquivo está em local correto (domain/service)
✅ DTOs em inglês, API em português
✅ Mappers transformam português ↔ inglês
✅ Todos os textos com useTranslation
✅ Constantes bem nomeadas e organizadas
✅ Context ao invés de prop-drilling
✅ Componentes pequenos e focados (< 100 linhas)
✅ Testes inclusos e passando
✅ Descrição clara da PR
```

---

## 📞 Dúvidas?

Se tiver dúvida sobre o padrão:

1. Consulte [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md)
2. Veja exemplos em `app/domain/` ou `app/service/`
3. Pergunte no Slack #dev
4. Abra discussion no GitHub

---

**Versão:** 1.0  
**Data:** Fevereiro 2026  
**Status:** ✅ Ativo e em Uso
