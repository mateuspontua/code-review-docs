# 🎬 Exemplos Práticos de Uso - Code Review Guide

## 📋 Cenário 1: Revisar uma PR Simples (5 minutos)

### Situação
Uma PR chega: "Add user card component"

### Passo 1: Leia a Descrição (1 min)
```
Title: Add user card component
Description:
- New reusable UserCard component
- Takes user object as prop
- Shows name, email, avatar
- Files: 3 alterados (UserCard.tsx, index.ts, test)
```

### Passo 2: Consulte Quick Reference (1 min)
```
Abra: CODE_REVIEW_QUICK_REFERENCE.md
Seção: "Quando Aprovar"

Checklist:
✅ Escopo bem definido? → Sim (apenas componente)
✅ < 15 arquivos? → Sim (3 arquivos)
✅ Sem magic numbers? → Preciso verificar
✅ Sem hardcoded? → Preciso verificar
```

### Passo 3: Revise o Código (3 min)

```typescript
// ❌ ENCONTRADO: Texto hardcoded
export const UserCard: React.FC<{ user: User }> = ({ user }) => (
    <div className="card">
        <h2>{user.name}</h2>
        <p>Email:</p>  {/* ← Hardcoded! */}
        <p>{user.email}</p>
    </div>
);

// ❌ ENCONTRADO: Magic number
<div style={{ maxWidth: 280 }}>  {/* ← Por que 280? */}
```

### Passo 4: Deixe Comentário Construtivo

```markdown
**Sugestão 1:** Traduzir "Email:"

**Contexto:** Segundo [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md),
seção "Tradução (i18n)", nenhum texto deve estar hardcoded.

**Código sugerido:**
```typescript
import { useTranslation } from 'react-i18next';

export const UserCard: React.FC<{ user: User }> = ({ user }) => {
    const [t] = useTranslation('translation');
    return (
        <div className="card">
            <h2>{user.name}</h2>
            <p>{t('common.labels.email')}</p>  {/* ✅ Traduzido */}
            <p>{user.email}</p>
        </div>
    );
};
```

---

**Sugestão 2:** Extrair constante para largura

**Contexto:** Segundo [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md),
seção "Magic Numbers", números isolados devem ter significado.

**Código sugerido:**
```typescript
const USER_CARD_CONFIG = {
    MAX_WIDTH: 280, // 14rem, padrão de design
} as const;

<div style={{ maxWidth: USER_CARD_CONFIG.MAX_WIDTH }}>
```

**Por quê:** Facilita reutilizar em outros lugares e deixa claro 
qual é o padrão de design para cards de usuário.
```

### Passo 5: Tome Decisão

```
Resultado: REQUEST CHANGES

Motivo: Texto hardcoded detectado (crítico por padrão)
Próximo: Autor corrige, você revisa novamente
```

---

## 📋 Cenário 2: Revisar uma PR Grande (Rejeitar)

### Situação
PR chega: "Refactor users domain"

### Primeiro: Verifique o Tamanho
```
FILES CHANGED: 32
INSERTIONS: 1200
DELETIONS: 300
```

### Ação Imediata: REQUEST CHANGES

```markdown
**Problema:** Esta PR é muito grande (32 arquivos alterados)

**Referência:** [CODE_REVIEW_QUICK_REFERENCE.md](./CODE_REVIEW_QUICK_REFERENCE.md)
seção "Tamanho de PR"

**Recomendação:**
✅ Ideal: 1-3 arquivos
✅ Aceitável: 4-8 arquivos
✅ Alerta: 9-15 arquivos
❌ Rejeitar: > 15 arquivos (32 é muito!)

**Sugestão:** Divida em múltiplas PRs menores:
1. PR 1: Criar domain/users/schemas (validators)
2. PR 2: Criar service/core/users (DTOs, mappers)
3. PR 3: Criar domain/users/entities (business logic)
4. PR 4: Criar domain/users/useCases (orchestration)
5. PR 5: Criar domain/users/presentation (hooks, components)

Isso permite:
- Reviews mais rápidas
- Menos chance de bugs
- Mais fácil de entender
- Fácil de fazer rollback se necessário
```

---

## 📋 Cenário 3: Revisar SOLID Violation

### Situação
PR com componente que faz muita coisa:

```typescript
function OrderForm() {
    // Estado do form
    const [formData, setFormData] = useState({});
    
    // Estado de fetch
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);
    
    // Estado de UI
    const [showModal, setShowModal] = useState(false);
    
    // Fetch data
    useEffect(() => { /* ... */ }, []);
    
    // Fetch mais dados
    useEffect(() => { /* ... */ }, []);
    
    // Validação
    const validateForm = () => { /* ... */ };
    
    // Submit
    const handleSubmit = async () => { /* ... */ };
    
    // 300 linhas de JSX...
    return (
        // Muita coisa aqui
    );
}
```

### Comentário:

```markdown
**Problema:** Esta função faz muitas coisas (SRP violation)

**Referência:** [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md),
seção "SOLID - Single Responsibility Principle"

**Responsabilidades identificadas:**
1. Gerenciar estado do form
2. Fetch de dados
3. Gerenciar UI (modal)
4. Validação
5. Submit

**Sugestão:** Separar em custom hooks:

```typescript
// 1. useOrderForm.ts - apenas estado do form
function useOrderForm() {
    const [formData, setFormData] = useState({});
    const validateForm = () => { /* ... */ };
    return { formData, setFormData, validateForm };
}

// 2. useFetchOrderData.ts - apenas fetch
function useFetchOrderData() {
    const [data, setData] = useState(null);
    const [loading, setLoading] = useState(false);
    const [error, setError] = useState(null);
    useEffect(() => { /* ... */ }, []);
    return { data, loading, error };
}

// 3. Componente fica limpo
function OrderForm() {
    const form = useOrderForm();
    const { data, loading } = useFetchOrderData();
    const [showModal, setShowModal] = useState(false);
    
    return (
        // Muito mais legível
    );
}
```

**Por quê:**
- Cada hook faz UMA coisa
- Hooks são reutilizáveis
- Componente é testável
- Fácil de manter
```

---

## 📋 Cenário 4: Revisar Domain/Service Pattern

### Situação
PR implementa novo domain: "Products"

### Checklist SERVICE LAYER:

```markdown
**Analisando:** app/service/core/products/

✅ Estrutura correta?
  ├─ ✅ dto/products.request.dto.ts
  ├─ ✅ dto/products.response.dto.ts
  ├─ ✅ interfaces/products.interface.ts
  ├─ ✅ mappers/products.mapper.ts
  ├─ ✅ services/http/products.service.ts
  ├─ ✅ services/http-client/products.service.ts
  └─ ✅ index.ts (barrel exports)

✅ DTOs em inglês?
  ├─ ✅ productId (não produtoId)
  ├─ ✅ name (não nome)
  └─ ✅ createdAt (não dataCriacao)

✅ Mappers transformam português ↔ inglês?
  ├─ ✅ API retorna: { produtoId, nome, dataCriacao }
  ├─ ✅ Mapper transforma: { productId, name, createdAt }
  └─ ✅ Implementação correta

✅ Há separação Server-Side e Client-Side?
  ├─ ✅ ServerSideProductsService em services/http/
  ├─ ✅ ClientSideProductsService em services/http-client/
  └─ ✅ Ambas implementam ProductsInterface

RESULTADO: ✅ APPROVE
```

### Checklist DOMAIN LAYER:

```markdown
**Analisando:** app/domain/catalog/products/

✅ Estrutura correta?
  ├─ ✅ entities/product.entity.ts
  ├─ ✅ dtos/ (para cada useCase)
  ├─ ✅ useCases/ (orquestração)
  ├─ ✅ validators/ (Zod schemas)
  ├─ ✅ presentation/ (hooks, components)
  └─ ✅ index.ts (barrel exports)

✅ Entities com lógica de negócio pura?
  ├─ ✅ Sem chamadas de API
  ├─ ✅ Sem side effects
  ├─ ✅ Apenas regras de negócio

✅ UseCases orquestram corretamente?
  ├─ ✅ GetProductsUseCase → service.getAll() → mapping → return
  ├─ ✅ CreateProductUseCase → validate → service.create() → return
  └─ ✅ Sem lógica de renderização

✅ Validators usam Zod?
  ├─ ✅ product-schema.ts define schema com Zod
  ├─ ✅ Validação antes de salvar
  └─ ✅ Mensagens de erro claras

✅ Hooks customizados abstraem complexidade?
  ├─ ✅ useGetProducts (invoca useCase + React Query)
  ├─ ✅ useCreateProduct (invoca useCase + mutation)
  └─ ✅ Componentes apenas chamam hooks

RESULTADO: ✅ APPROVE
```

---

## 📋 Cenário 5: Revisar Props-Drilling Excessivo

### Situação
PR com muitas props sendo repassadas:

```typescript
<Page 
    user={user}
    theme={theme}
    onUserUpdate={onUserUpdate}
    onThemeChange={onThemeChange}
    locale={locale}
    apiBaseUrl={apiBaseUrl}
    errorHandler={errorHandler}
/>

function Page(props: {
    user: User;
    theme: Theme;
    onUserUpdate: ...;
    // ... tudo repetindo para UserForm
}) {
    return <UserForm {...props} />;
}

function UserForm(props: {
    user: User;
    theme: Theme;
    onUserUpdate: ...;
    // Mas UserForm só usa user e onUserUpdate!
    theme={props.theme} // Repassa mas não usa
    onThemeChange={props.onThemeChange} // Repassa mas não usa
}) {
    // ...
}
```

### Comentário:

```markdown
**Problema:** Prop-drilling excessivo (6+ props sendo repassadas)

**Referência:** [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md),
seção "Props e Prop-Drilling"

**Sugestão:** Usar Context API

**Implementação:**
```typescript
// contexts/AppContext.ts
export const AppContext = createContext<{
    user: User;
    theme: Theme;
    onUserUpdate: (user: User) => void;
    onThemeChange: (theme: Theme) => void;
    locale: string;
    apiBaseUrl: string;
    errorHandler: (error: Error) => void;
} | null>(null);

export function useAppContext() {
    const ctx = useContext(AppContext);
    if (!ctx) throw new Error('Usar dentro de AppProvider');
    return ctx;
}

// App.tsx
<AppContext.Provider value={appValues}>
    <Page />
</AppContext.Provider>

// Page.tsx - mais limpa!
function Page() {
    return <UserForm />;
}

// UserForm.tsx - pega direto do context
function UserForm() {
    const { user, onUserUpdate } = useAppContext();
    return (
        <form>
            <input value={user.name} onChange={...} />
        </form>
    );
}
```

**Por quê:**
- Componentes intermediários não precisam das props
- Mais fácil adicionar novos contextos
- Code é mais legível
```

---

## 📋 Cenário 6: Reveiew Performance Issues

### Situação
PR com problema de performance:

```typescript
// ❌ PROBLEMA 1: Sem memo()
export const UserCard = ({ user }: { user: User }) => {
    return (
        <div>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
};

// ❌ PROBLEMA 2: Sem useMemo
function UserList({ users }: { users: User[] }) {
    const filtered = users.filter(u => u.active); // Recalcula sempre!
    return (
        <div>
            {filtered.map(u => (
                <UserCard user={u} />  {/* Sem memo, renderiza sempre */}
            ))}
        </div>
    );
}

// ❌ PROBLEMA 3: Sem useCallback
function UserListContainer() {
    const handleClick = () => { /* ... */ }; // Nova função sempre!
    return <UserList onUserClick={handleClick} />;
}
```

### Comentário:

```markdown
**Observação:** Oportunidades de otimização detectadas

**Referência:** [CODE_REVIEW_GUIDE.md](./CODE_REVIEW_GUIDE.md),
seção "Renderizações Desnecessárias"

**Melhorias sugeridas:**

```typescript
// 1. UserCard com memo()
export const UserCard = React.memo(({ user }: { user: User }) => {
    return (
        <div>
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
});

// 2. Filtro com useMemo
function UserList({ users }: { users: User[] }) {
    const filtered = useMemo(() => 
        users.filter(u => u.active),
        [users]
    );
    
    return (
        <div>
            {filtered.map(u => (
                <UserCard key={u.id} user={u} />
            ))}
        </div>
    );
}

// 3. Callback com useCallback
function UserListContainer() {
    const handleClick = useCallback(() => {
        /* ... */
    }, []); // Mesma função sempre
    
    return <UserList onUserClick={handleClick} />;
}
```

**Impacto:** Reduz re-renders desnecessários, melhora responsividade
```

---

## 🎯 Resumo de Uso

| Cenário | Tempo | Ação | Resultado |
|---------|-------|------|-----------|
| PR simples | 5 min | Quick ref + review | Approve ou comment |
| PR grande | 2 min | Quick ref | Request changes |
| SOLID violation | 10 min | Consultar guia | Request changes + sugestão |
| Domain/Service | 15 min | Checklists | Approve |
| Props drilling | 10 min | Consultar guia | Request changes |
| Performance | 5 min | Sugestão | Comment (não crítico) |

---

**Versão:** 1.0  
**Data:** Fevereiro 2026
