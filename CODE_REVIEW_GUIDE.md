# 📋 Guia de Code Review para Pull Requests - Pontua

**Versão:** 1.0  
**Data:** Fevereiro 2026  
**Propósito:** Estabelecer um padrão coeso de revisão de código para garantir qualidade, manutenibilidade e consistência do projeto.

---

## Índice

1. [Visão Geral](#visão-geral)
2. [Antes de Fazer Code Review](#antes-de-fazer-code-review)
3. [Checklist de Código](#checklist-de-código)
4. [Boas Práticas por Contexto](#boas-práticas-por-contexto)
5. [Checklist de Arquitetura](#checklist-de-arquitetura)
6. [Checklist de Performance](#checklist-de-performance)
7. [Checklist de Acessibilidade e UX](#checklist-de-acessibilidade-e-ux)
8. [Como Fazer um Bom Comentário](#como-fazer-um-bom-comentário)
9. [Quando Rejeitar ou Solicitar Mudanças](#quando-rejeitar-ou-solicitar-mudanças)

---

## Visão Geral

Este guia é um **manual de referência** para todos os desenvolvedores que fazem code review em Pull Requests. O objetivo é garantir que o código adicionado ao projeto:

- ✅ Siga os **princípios SOLID** (quando aplicável ao frontend)
- ✅ Adira à **arquitetura Domain/Service** do projeto
- ✅ Use o sistema de **tradução corretamente** (i18n)
- ✅ Evite **anti-patterns** comuns (magic numbers, prop-drilling, etc)
- ✅ Respeite **boas práticas de React** (memo, lazy, hooks)
- ✅ Tenha um **tamanho razoável** (não muitos arquivos)
- ✅ Seja **testável** e **manutenível**

**Dica:** Ao revisar, pergunte-se: *"Eu conseguiria entender e manter este código daqui a 6 meses?"*

---

## Antes de Fazer Code Review

### 1. Contextualize a PR

Leia a **descrição da PR** e o **tipo de mudança**:

- [ ] É um bugfix?
- [ ] É uma nova feature?
- [ ] É um refator?
- [ ] É documentação?

### 2. Entenda o Escopo

Verifique se a PR está **bem delimitada**:

- [ ] A PR trata de apenas um problema ou uma funcionalidade?
- [ ] Há commits não relacionados ao escopo principal?
- [ ] O número de arquivos alterados é razoável? (vide [Seção "Tamanho da PR"](#tamanho-da-pr))

### 3. Leia o Histórico de Commits

Os commits devem ser **atômicos** e **bem nomeados**:

- [ ] Cada commit representa uma mudança lógica?
- [ ] As mensagens de commit seguem a convenção do projeto? (e.g., `feat:`, `fix:`, `refactor:`)
- [ ] Não há commits com mensagens genéricas como `"fix"` ou `"update"`?

### 4. Verifique a Branch de Destino

- [ ] A PR está sendo mergeada na branch correta? (e.g., `develop` ou `main`)
- [ ] Não há conflitos com a branch de destino?

---

## Checklist de Código

### SOLID - Princípios Fundamentais

#### Single Responsibility Principle (SRP)

Cada arquivo/função deve ter **uma única razão para mudar**.

**❌ Errado:**
```typescript
// Este arquivo faz muita coisa
function UserComponent() {
    const [users, setUsers] = useState([]);
    const [filter, setFilter] = useState('');
    const [sortBy, setSortBy] = useState('name');
    
    // Fetch
    useEffect(() => { fetchUsers(); }, []);
    
    // Filter
    const filtered = users.filter(u => u.name.includes(filter));
    
    // Sort
    const sorted = filtered.sort((a, b) => a[sortBy].localeCompare(b[sortBy]));
    
    // Render
    return (
        <div>
            <input onChange={(e) => setFilter(e.target.value)} />
            <select onChange={(e) => setSortBy(e.target.value)} >...</select>
            <Table data={sorted} />
        </div>
    );
}
```

**✅ Correto:**
```typescript
// useUserFiltering.ts - responsável apenas por filtrar
function useUserFiltering(users: User[], searchTerm: string) {
    return useMemo(() => 
        users.filter(u => u.name.includes(searchTerm)), 
        [users, searchTerm]
    );
}

// useUserSorting.ts - responsável apenas por ordenar
function useUserSorting(users: User[], sortBy: string) {
    return useMemo(() => 
        [...users].sort((a, b) => a[sortBy].localeCompare(b[sortBy])), 
        [users, sortBy]
    );
}

// UserComponent.tsx - responsável apenas por renderizar
function UserComponent() {
    const [filter, setFilter] = useState('');
    const [sortBy, setSortBy] = useState('name');
    const users = useGetUsers();
    
    const filtered = useUserFiltering(users, filter);
    const sorted = useUserSorting(filtered, sortBy);
    
    return (
        <div>
            <UserFilterInput value={filter} onChange={setFilter} />
            <UserSortSelect value={sortBy} onChange={setSortBy} />
            <Table data={sorted} />
        </div>
    );
}
```

**Pontos a verificar:**
- [ ] Componentes fazem uma coisa bem?
- [ ] Lógica de estado está separada da lógica de renderização?
- [ ] Hooks customizados têm responsabilidades claras?

---

#### Open/Closed Principle (OCP)

Código deve estar **aberto para extensão, fechado para modificação**.

**❌ Errado:**
```typescript
// Sempre que houver um novo tipo de badge, precisa modificar este código
function StatusBadge({ status }: { status: string }) {
    if (status === 'active') return <span className="bg-green">Ativo</span>;
    if (status === 'pending') return <span className="bg-yellow">Pendente</span>;
    if (status === 'inactive') return <span className="bg-red">Inativo</span>;
}
```

**✅ Correto:**
```typescript
// Define um mapa extensível
const STATUS_CONFIG = {
    active: { label: 'Ativo', className: 'bg-green' },
    pending: { label: 'Pendente', className: 'bg-yellow' },
    inactive: { label: 'Inativo', className: 'bg-red' },
} as const;

function StatusBadge({ status }: { status: keyof typeof STATUS_CONFIG }) {
    const config = STATUS_CONFIG[status];
    return <span className={config.className}>{config.label}</span>;
}
```

**Pontos a verificar:**
- [ ] Há switch-cases ou if-elses longos que poderiam ser data-driven?
- [ ] Existe um mapa de configuração que facilita extensão?
- [ ] Novas features requerem modificação de código existente?

---

#### Liskov Substitution Principle (LSP)

Subclasses devem poder substituir suas superclasses sem quebrar.

**❌ Errado:**
```typescript
interface Animal {
    move(): void;
}

class Bird implements Animal {
    move() { /* fly */ }
}

class Penguin implements Animal {
    move() { /* swim */ }
    // Pinguim não voa, mas implementa a interface Bird
}
```

**✅ Correto:**
```typescript
interface Animal {
    move(): void;
}

interface FlyingAnimal extends Animal {
    fly(): void;
}

class Bird implements FlyingAnimal {
    move() { /* fly */ }
    fly() { /* fly */ }
}

class Penguin implements Animal {
    move() { /* swim */ }
    // Pinguim só implementa o que consegue fazer
}
```

**Pontos a verificar:**
- [ ] Interfaces e tipos refletem comportamentos reais?
- [ ] Há implementações que violam o contrato da interface?
- [ ] Componentes/hooks esperam comportamentos que nem sempre existem?

---

#### Interface Segregation Principle (ISP)

Cliente não deve depender de interfaces que não usa.

**❌ Errado:**
```typescript
interface UserService {
    getUser(): User;
    updateUser(user: User): void;
    deleteUser(userId: string): void;
    sendEmail(to: string, body: string): void; // Não relacionado!
    generateReport(): Report; // Não relacionado!
}

class MyComponent {
    constructor(private userService: UserService) {}
    
    handleDelete(userId: string) {
        this.userService.deleteUser(userId);
        // Mas tenho acesso a sendEmail e generateReport que não preciso
    }
}
```

**✅ Correto:**
```typescript
interface UserRepository {
    getUser(): User;
    updateUser(user: User): void;
    deleteUser(userId: string): void;
}

interface EmailService {
    sendEmail(to: string, body: string): void;
}

interface ReportGenerator {
    generateReport(): Report;
}

class MyComponent {
    constructor(private userRepo: UserRepository) {}
    
    handleDelete(userId: string) {
        this.userRepo.deleteUser(userId);
        // Apenas o que preciso
    }
}
```

**Pontos a verificar:**
- [ ] Props de componentes têm apenas o que é necessário?
- [ ] Interfaces são pequenas e focadas?
- [ ] Há "props gigantescas" passadas por prop-drilling?

---

#### Dependency Inversion Principle (DIP)

Código de alto nível não deve depender de código de baixo nível. Ambos devem depender de abstrações.

**❌ Errado:**
```typescript
import axios from 'axios'; // Dependência direta

class UserRepository {
    async getUser(id: string) {
        const response = await axios.get(`/users/${id}`);
        return response.data;
    }
}
```

**✅ Correto:**
```typescript
interface HttpClient {
    get<T>(url: string): Promise<T>;
}

class UserRepository {
    constructor(private httpClient: HttpClient) {}
    
    async getUser(id: string) {
        return this.httpClient.get(`/users/${id}`);
    }
}
```

**Pontos a verificar:**
- [ ] Há imports diretos de bibliotecas em múltiplos lugares?
- [ ] Seria fácil trocar uma dependência (e.g., axios → fetch)?
- [ ] O código está acoplado a implementações específicas?

---

### Domain/Service Pattern

Toda PR que toca em `app/domain/` ou `app/service/` **DEVE** seguir o documento [DOMAIN_SERVICE_PATTERN.md](./DOMAIN_SERVICE_PATTERN.md).

#### ✅ Checklist Service Layer

- [ ] Service em `app/service/core/[modulo]/` segue a estrutura (dto, interfaces, mappers, services)?
- [ ] DTOs em inglês (normalizados), API em português (mapeados)?
- [ ] Mappers transformam português → inglês (e vice-versa para requests)?
- [ ] Há separação entre Server-Side e Client-Side?
- [ ] Todas as implementações usam a mesma interface?
- [ ] Não há lógica de negócio no service (apenas comunicação com API)?

#### ✅ Checklist Domain Layer

- [ ] Domain em `app/domain/[categoria]/[modulo]/` segue a estrutura?
- [ ] Entities com lógica de negócio pura?
- [ ] UseCases orquestram o fluxo de negócio?
- [ ] Validators usam Zod para schema validation?
- [ ] Factories criam instâncias com dependências injetadas?
- [ ] Hooks customizados abstraem a complexidade de UseCases?
- [ ] Não há lógica de negócio em componentes React?

---

### Tradução (i18n)

**Regra de Ouro:** Nenhum texto em português ou inglês deve estar "hardcoded" no código. Use `useTranslation`.

#### ✅ Checklist Tradução

**❌ Errado:**
```typescript
function UserForm() {
    return (
        <div>
            <label>Nome do Usuário</label> {/* Hardcoded! */}
            <input placeholder="Digite aqui..." /> {/* Hardcoded! */}
            <button>Salvar</button> {/* Hardcoded! */}
        </div>
    );
}
```

**✅ Correto:**
```typescript
import { useTranslation } from 'react-i18next';

function UserForm() {
    const [t] = useTranslation('translation');
    
    return (
        <div>
            <label>{t('pages.userForm.labels.username')}</label>
            <input placeholder={t('pages.userForm.placeholders.username')} />
            <button>{t('common.buttons.save')}</button>
        </div>
    );
}
```

**Pontos a verificar:**
- [ ] Nenhum texto em português/inglês sem `t()`?
- [ ] `useTranslation` é importado de `react-i18next`?
- [ ] Chaves de tradução são específicas e descritivas? (não genéricas)
- [ ] Estão em `app/translate/locales/ts/pt.br.ts`?
- [ ] O tipo está em `app/translate/translations.d.ts`?

---

### Magic Numbers e Valores Hardcoded

**Regra:** Números sem contexto ("magic numbers") devem ser evitados. Use constantes com nomes significativos.

#### ✅ Checklist Magic Numbers

**❌ Errado:**
```typescript
function UserTable({ users }: { users: User[] }) {
    const visibleUsers = users.slice(0, 50); // Por que 50?
    
    return (
        <table>
            {visibleUsers.map(user => (
                <tr key={user.id}>
                    <td style={{ width: 120 }}>...</td> {/* Por que 120? */}
                    <td>{user.email}</td>
                </tr>
            ))}
        </table>
    );
}

const debounceDelay = 300; // Por que 300ms?
```

**✅ Correto:**
```typescript
const TABLE_CONFIG = {
    DEFAULT_PAGE_SIZE: 50,
    COLUMN_WIDTH: {
        NAME: 120,
        EMAIL: 200,
    },
} as const;

const DEBOUNCE_DELAYS = {
    SEARCH: 300,
    RESIZE: 200,
} as const;

function UserTable({ users }: { users: User[] }) {
    const visibleUsers = users.slice(0, TABLE_CONFIG.DEFAULT_PAGE_SIZE);
    
    return (
        <table>
            {visibleUsers.map(user => (
                <tr key={user.id}>
                    <td style={{ width: TABLE_CONFIG.COLUMN_WIDTH.NAME }}>...</td>
                    <td>{user.email}</td>
                </tr>
            ))}
        </table>
    );
}

const searchDebounce = useDebounce(searchTerm, DEBOUNCE_DELAYS.SEARCH);
```

**Pontos a verificar:**
- [ ] Números isolados têm um nome e contexto?
- [ ] Configurações estão em objetos `*_CONFIG`?
- [ ] Constantes são reutilizáveis?
- [ ] Há comentário explicando o "por quê" se não óbvio?

---

### Props e Prop-Drilling

Componentes não devem ter muitas props ou repassar props desnecessariamente.

#### ✅ Checklist Props

**❌ Errado:**
```typescript
function Page(props: {
    user: User;
    theme: Theme;
    onUserUpdate: (user: User) => void;
    onThemeChange: (theme: Theme) => void;
    locale: string;
    apiBaseUrl: string;
    errorHandler: (error: Error) => void;
}) {
    return (
        <UserSection
            user={props.user}
            theme={props.theme}
            onUserUpdate={props.onUserUpdate}
            onThemeChange={props.onThemeChange}
            locale={props.locale}
            apiBaseUrl={props.apiBaseUrl}
            errorHandler={props.errorHandler}
        />
    );
}

function UserSection(props: any) { // muitos props!
    return (
        <UserForm
            user={props.user}
            theme={props.theme}
            onUserUpdate={props.onUserUpdate}
            // ... repetindo props
        />
    );
}
```

**✅ Correto (com Context):**
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
    if (!ctx) throw new Error('useAppContext deve estar dentro de AppProvider');
    return ctx;
}

// Componente raiz
function App() {
    const appContextValue = { /* ... */ };
    return (
        <AppContext.Provider value={appContextValue}>
            <Page />
        </AppContext.Provider>
    );
}

// Agora Page não precisa receber todas as props
function Page() {
    return <UserSection />;
}

function UserSection() {
    return <UserForm />;
}

function UserForm() {
    const { user, onUserUpdate } = useAppContext();
    return (
        <form>
            <input value={user.name} onChange={(e) => onUserUpdate({ ...user, name: e.target.value })} />
        </form>
    );
}
```

**Pontos a verificar:**
- [ ] Componentes têm mais de 5-7 props diferentes?
- [ ] Props estão sendo repassadas sem serem usadas (prop-drilling)?
- [ ] Seria melhor usar Context API ou estado global?
- [ ] Componentes "containers" têm props demais?

**Exceção:** Props de estilo e comportamento básicos (className, disabled, etc) são aceitáveis.

---

### Conditional Rendering (Se/Senão)

Evite switch-cases e if-elses longos. Use data-driven ou estratégia de componentes.

#### ✅ Checklist Conditionals

**❌ Errado:**
```typescript
function StatusIcon({ status }: { status: string }) {
    if (status === 'active') {
        return <CheckCircleIcon color="green" />;
    } else if (status === 'pending') {
        return <ClockIcon color="orange" />;
    } else if (status === 'inactive') {
        return <XCircleIcon color="red" />;
    } else if (status === 'blocked') {
        return <LockIcon color="darkRed" />;
    } else {
        return <QuestionIcon color="gray" />;
    }
}

// Ou piores ainda:
function OrderStatus({ order }: { order: Order }) {
    switch (order.status) {
        case 'pending':
            return <PendingOrderComponent order={order} />;
        case 'processing':
            return <ProcessingOrderComponent order={order} />;
        case 'shipped':
            return <ShippedOrderComponent order={order} />;
        case 'delivered':
            return <DeliveredOrderComponent order={order} />;
        default:
            return <UnknownOrderComponent />;
    }
}
```

**✅ Correto:**
```typescript
const STATUS_ICON_MAP = {
    active: { Icon: CheckCircleIcon, color: 'green' },
    pending: { Icon: ClockIcon, color: 'orange' },
    inactive: { Icon: XCircleIcon, color: 'red' },
    blocked: { Icon: LockIcon, color: 'darkRed' },
} as const;

function StatusIcon({ status }: { status: keyof typeof STATUS_ICON_MAP }) {
    const { Icon, color } = STATUS_ICON_MAP[status] ?? { 
        Icon: QuestionIcon, 
        color: 'gray' 
    };
    return <Icon color={color} />;
}

// Para componentes complexos, use factories:
const STATUS_COMPONENT_FACTORY = {
    pending: PendingOrderComponent,
    processing: ProcessingOrderComponent,
    shipped: ShippedOrderComponent,
    delivered: DeliveredOrderComponent,
} as const;

function OrderStatus({ order }: { order: Order }) {
    const Component = STATUS_COMPONENT_FACTORY[order.status] ?? UnknownOrderComponent;
    return <Component order={order} />;
}
```

**Pontos a verificar:**
- [ ] Há switch-cases ou if-elses com muitos branches (> 3)?
- [ ] Poderia usar um mapa (object/Map)?
- [ ] Há lógica condicional que se repete em vários lugares?
- [ ] Seria mais legível com um componente "estratégia"?

---

### React Hooks - memo() e lazy()

Use `React.memo()` e `React.lazy()` corretamente para otimizar performance.

#### ✅ Checklist memo()

**Quando usar memo():**
- Componente é "puro" (mesmo props → mesmo resultado)?
- Componente é renderizado frequentemente com props iguais?
- Componente é pesado (muitas renderizações)?

**❌ Errado:**
```typescript
// Usa memo de forma desnecessária
export const SimpleText = React.memo(({ text }: { text: string }) => {
    return <p>{text}</p>; // Componente muito simples, sem lógica
});

// Usa memo mas props mudam sempre (não aproveita)
function UserForm() {
    const handleSubmit = () => { /* ... */ }; // Nova função a cada render
    
    return <Form onSubmit={handleSubmit} />;
}

export const Form = React.memo(({ onSubmit }: { onSubmit: () => void }) => {
    return <form onSubmit={onSubmit}>...</form>;
});
```

**✅ Correto:**
```typescript
// memo para componente que renderiza frequentemente com props iguais
export const UserCard = React.memo(({ user }: { user: User }) => {
    return (
        <div className="card">
            <h2>{user.name}</h2>
            <p>{user.email}</p>
        </div>
    );
});

// Se props mudam sempre, use useCallback para a função
function UserForm() {
    const handleSubmit = useCallback(() => {
        // lógica
    }, []); // Dependências vazias = mesma função sempre
    
    return <Form onSubmit={handleSubmit} />;
}

export const Form = React.memo(({ onSubmit }: { onSubmit: () => void }) => {
    return <form onSubmit={onSubmit}>...</form>;
});
```

**Pontos a verificar:**
- [ ] memo() é usado sem justificativa (componentes simples)?
- [ ] Props passadas para memo() mudam frequentemente?
- [ ] Há callbacks/objetos não memoizados passando para componentes memo()?
- [ ] Há muitos re-renders desnecessários? (pode validar com React DevTools Profiler)

---

#### ✅ Checklist lazy()

**Quando usar lazy():**
- Route que não é crítica no carregamento inicial?
- Componente pesado que não aparece acima do fold?
- Modal ou feature que usuário pode não usar?

**❌ Errado:**
```typescript
// Tudo lazy (até componentes críticos)
const Header = React.lazy(() => import('./Header'));
const Navigation = React.lazy(() => import('./Navigation'));
const Footer = React.lazy(() => import('./Footer'));

export default function App() {
    return (
        <Suspense fallback={<Loader />}>
            <Header />
            <Navigation />
            <Main />
            <Footer />
        </Suspense>
    );
}

// Ou nunca usar lazy quando deveria
import HeavyReportComponent from './HeavyReport'; // 500KB!
import HeavyChartComponent from './HeavyChart'; // 300KB!

function Dashboard() {
    return (
        <div>
            <Overview />
            <HeavyReportComponent /> {/* Carrega sempre, mas usuário nem vê */}
            <HeavyChartComponent />
        </div>
    );
}
```

**✅ Correto:**
```typescript
// Componentes críticos: importa normal
import Header from './Header';
import Navigation from './Navigation';

// Componentes não-críticos: lazy
const Footer = React.lazy(() => import('./Footer'));
const ReportModal = React.lazy(() => import('./ReportModal'));

export default function App() {
    return (
        <div>
            <Header />
            <Navigation />
            <Main />
            <Suspense fallback={<FooterSkeleton />}>
                <Footer />
            </Suspense>
        </div>
    );
}

function Dashboard() {
    const [showReport, setShowReport] = useState(false);
    
    return (
        <div>
            <Overview />
            {showReport && (
                <Suspense fallback={<ReportSkeleton />}>
                    <ReportModal onClose={() => setShowReport(false)} />
                </Suspense>
            )}
        </div>
    );
}
```

**Pontos a verificar:**
- [ ] Componentes críticos (Header, Nav) estão lazy? (não devem estar)
- [ ] Componentes pesados que aparece depois estão lazy?
- [ ] Há Suspense fallback apropriado?
- [ ] O bundle size diminuiu? (pode validar com `npm run build --analyze`)

---

### Tamanho da PR

Uma PR não deve alterar **muitos arquivos** ou ser **muito grande**.

#### ✅ Checklist Tamanho

**Referências:**
- ✅ **Ideal:** 1-3 arquivos alterados
- ✅ **Aceitável:** 4-8 arquivos alterados
- ⚠️ **Alerta:** 9-15 arquivos alterados (questione se é necessário)
- ❌ **Rejeitar:** > 15 arquivos alterados em uma única PR

**❌ Errado:**
```
files changed: 45
insertions: 2341
deletions: 890

Exemplos:
- Adicionar feature nova + refatorar 3 módulos + corrigir styles globais
- Migrar lib X para lib Y (muitos arquivos)
- Criar novo domain completo com 20+ arquivos
```

**✅ Correto:**
```
files changed: 6
insertions: 234
deletions: 45

Exemplos:
- Criar novo domain completo: 1 PR (bem estruturada)
- Bugfix em um módulo: 1-3 arquivos
- Feature pequena: 3-5 arquivos
```

**Se PR é muito grande:**
1. Divida em múltiplas PRs menores
2. Cada PR deve ter um escopo claro
3. Exemplo:
   - PR #1: Criar domain/schemas para validação
   - PR #2: Criar service/core com DTOs e mappers
   - PR #3: Implementar useCases
   - PR #4: Criar componentes UI
   - PR #5: Integrar tudo

**Pontos a verificar:**
- [ ] PR alterou quantos arquivos? (deve ter justificativa se > 10)
- [ ] É possível dividir em PRs menores?
- [ ] Há commits não relacionados ao escopo?
- [ ] A descrição da PR explica por que tantos arquivos?

---

## Boas Práticas por Contexto

### Componentes React

#### Type-Safety

```typescript
// ✅ Correto: tipos explícitos
interface ButtonProps {
    label: string;
    onClick: () => void;
    variant?: 'primary' | 'secondary';
    disabled?: boolean;
}

export const Button: React.FC<ButtonProps> = ({ label, onClick, variant = 'primary', disabled = false }) => {
    return (
        <button className={`btn-${variant}`} onClick={onClick} disabled={disabled}>
            {label}
        </button>
    );
};

// ❌ Errado: tipos genéricos
interface ButtonProps {
    [key: string]: any;
}

export const Button: React.FC<ButtonProps> = (props: any) => {
    return <button {...props}>{props.label}</button>;
};
```

- [ ] Props têm tipos explícitos (não `any`)?
- [ ] Return type do componente é `React.FC` ou `JSX.Element`?
- [ ] Há `PropsWithChildren` quando apropriado?

#### Lógica em Hooks vs Componentes

```typescript
// ✅ Correto: lógica em hook
function useUserData(userId: string) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(false);

    useEffect(() => {
        setLoading(true);
        fetchUser(userId).then(setUser).finally(() => setLoading(false));
    }, [userId]);

    return { user, loading };
}

function UserProfile({ userId }: { userId: string }) {
    const { user, loading } = useUserData(userId);
    return loading ? <Loader /> : <div>{user?.name}</div>;
}

// ❌ Errado: lógica no componente
function UserProfile({ userId }: { userId: string }) {
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(false);

    useEffect(() => {
        setLoading(true);
        fetchUser(userId).then(setUser).finally(() => setLoading(false));
    }, [userId]);

    return loading ? <Loader /> : <div>{user?.name}</div>;
}
```

- [ ] Lógica complexa está em hooks customizados?
- [ ] Componentes são "dumb" (apenas renderizam)?
- [ ] Há duplicação de lógica que poderia ser um hook?

---

### UseState vs UseReducer

```typescript
// ✅ UseReducer para estado complexo
type UserState = { user: User | null; loading: boolean; error: Error | null };
type UserAction = 
    | { type: 'LOADING' }
    | { type: 'SUCCESS'; payload: User }
    | { type: 'ERROR'; payload: Error };

function userReducer(state: UserState, action: UserAction): UserState {
    switch (action.type) {
        case 'LOADING':
            return { ...state, loading: true };
        case 'SUCCESS':
            return { user: action.payload, loading: false, error: null };
        case 'ERROR':
            return { user: null, loading: false, error: action.payload };
    }
}

// ❌ Evitar: múltiplos useState para estado relacionado
const [user, setUser] = useState(null);
const [loading, setLoading] = useState(false);
const [error, setError] = useState(null);
```

- [ ] Há muitos `useState` relacionados? (considere `useReducer`)
- [ ] Há lógica complexa em updates de estado?
- [ ] Actions estão bem tipadas?

---

### Custom Hooks

```typescript
// ✅ Bom: hook é pequeno, reutilizável e testável
function useDebouncedValue<T>(value: T, delay: number): T {
    const [debouncedValue, setDebouncedValue] = useState(value);

    useEffect(() => {
        const handler = setTimeout(() => setDebouncedValue(value), delay);
        return () => clearTimeout(handler);
    }, [value, delay]);

    return debouncedValue;
}

// ❌ Errado: hook com muita lógica
function useUserDashboard(userId: string) {
    const [user, setUser] = useState(null);
    const [posts, setPosts] = useState([]);
    const [comments, setComments] = useState([]);
    const [stats, setStats] = useState(null);
    // 200 linhas de código...
}
```

- [ ] Hook é testável isoladamente?
- [ ] Hook é reutilizável em múltiplos componentes?
- [ ] Não está fazendo muitas responsabilidades?
- [ ] Dependencies array está completo e preciso?

---

## Checklist de Arquitetura

### Estrutura de Pastas

```
✅ Esperado:
app/
├── domain/
│   ├── [categoria]/
│   │   └── [modulo]/
│   │       ├── dtos/
│   │       ├── entities/
│   │       ├── useCases/
│   │       ├── validators/
│   │       ├── presentation/
│   │       └── index.ts
│   └── ...
└── service/
    └── core/
        └── [modulo]/
            ├── dto/
            ├── interfaces/
            ├── mappers/
            ├── services/
            └── index.ts

❌ Evitar:
- Criar pastas em locais aleatórios (utils, lib, etc)
- Misturar domain e service
- Componentes dentro de service
```

- [ ] Novos arquivos estão nos locais corretos?
- [ ] Há duplicação de código em pastas diferentes?
- [ ] Estrutura é consistente com projeto existente?

---

### Importes e Exports

```typescript
// ✅ Correto: imports limpos via index.ts (barrel exports)
import { UserEntity } from '@domain/people-management/users';
import { CreateUserUseCase } from '@domain/people-management/users';
import { useCreateUser } from '@domain/people-management/users';

// Arquivo: app/domain/people-management/users/index.ts
export { UserEntity } from './entities';
export { CreateUserUseCase } from './useCases';
export { useCreateUser } from './presentation/hooks';

// ❌ Errado: imports longos e diretos
import { UserEntity } from '@domain/people-management/users/entities/user.entity';
import { CreateUserUseCase } from '@domain/people-management/users/useCases/create-user.useCase';
import { useCreateUser } from '@domain/people-management/users/presentation/hooks/use-create-user.tsx';
```

- [ ] Imports usam caminhos curtos via index.ts?
- [ ] Há um index.ts em cada pasta principal?
- [ ] Exports estão bem organizados?

---

## Checklist de Performance

### Bundle Size

```typescript
// ✅ Correto: lazy loading de componentes pesados
const HeavyReport = React.lazy(() => import('./HeavyReport'));

function Dashboard() {
    const [showReport, setShowReport] = useState(false);
    
    return (
        <div>
            <button onClick={() => setShowReport(true)}>Gerar Relatório</button>
            {showReport && (
                <Suspense fallback={<Skeleton />}>
                    <HeavyReport />
                </Suspense>
            )}
        </div>
    );
}

// ❌ Errado: importar tudo que é pesado
import HeavyReport from './HeavyReport';
import HeavyChart from './HeavyChart';
import HeavyMap from './HeavyMap';

function Dashboard() {
    return (
        <div>
            <HeavyReport />
            <HeavyChart />
            <HeavyMap />
        </div>
    );
}
```

- [ ] Há imports desnecessários?
- [ ] Bibliotecas pesadas estão sendo lazy?
- [ ] Bundle size aumentou significativamente? (validar com `npm run build`)

---

### Renderizações Desnecessárias

```typescript
// ✅ Correto: useMemo e useCallback para evitar re-renders
function UserList({ users, onUserClick }: Props) {
    const filteredUsers = useMemo(() => 
        users.filter(u => u.active),
        [users]
    );

    const handleClick = useCallback((id: string) => {
        onUserClick(id);
    }, [onUserClick]);

    return (
        <div>
            {filteredUsers.map(u => (
                <UserCard key={u.id} user={u} onClick={() => handleClick(u.id)} />
            ))}
        </div>
    );
}

// ❌ Errado: recalcula e recria funções a cada render
function UserList({ users, onUserClick }: Props) {
    const filteredUsers = users.filter(u => u.active); // Recalcula sempre
    
    return (
        <div>
            {filteredUsers.map(u => (
                <UserCard 
                    key={u.id} 
                    user={u} 
                    onClick={() => onUserClick(u.id)} // Nova função sempre
                />
            ))}
        </div>
    );
}
```

- [ ] Há cálculos pesados sem `useMemo`?
- [ ] Callbacks/objetos estão sem `useCallback`/`useMemo`?
- [ ] Keys em listas são estáveis? (não use índice se lista muda)

---

### Query Optimization

```typescript
// ✅ Correto: paginar dados
function UserTable() {
    const [page, setPage] = useState(0);
    const { data, isLoading } = useQuery(['users', page], () => 
        userService.getAll({ page, limit: 50 })
    );
    
    return (
        <div>
            <Table data={data?.items} />
            <Pagination current={page} total={data?.total} onChange={setPage} />
        </div>
    );
}

// ❌ Errado: carregar tudo de uma vez
function UserTable() {
    const { data } = useQuery(['users'], () => userService.getAll()); // 10000 usuários!
    return <Table data={data} />;
}
```

- [ ] Queries retornam muitos dados de uma vez?
- [ ] Há paginação/virtualization quando necessário?
- [ ] Dados cacheados apropriadamente?

---

## Checklist de Acessibilidade e UX

### A11y (Acessibilidade)

```typescript
// ✅ Correto: semântico e acessível
<button 
    aria-label="Fechar diálogo"
    onClick={onClose}
>
    ✕
</button>

<form onSubmit={handleSubmit}>
    <label htmlFor="email">Email:</label>
    <input id="email" type="email" required />
    <button type="submit">Enviar</button>
</form>

// ❌ Errado: não acessível
<div onClick={onClose} role="button">✕</div>
<div>Email:</div>
<input />
<div onClick={handleSubmit}>Enviar</div>
```

- [ ] Form inputs têm labels?
- [ ] Buttons têm aria-labels se necessário?
- [ ] Componentes interativos usam tags semânticas (button, a, etc)?
- [ ] Contraste de cores adequado?
- [ ] Navegação por teclado funciona?

---

### UX

```typescript
// ✅ Correto: feedback visual para usuário
<button disabled={isLoading}>
    {isLoading ? 'Carregando...' : 'Salvar'}
</button>

<input placeholder={t('common.states.required')} required />

// ❌ Errado: sem feedback
<button onClick={submit}>Salvar</button>
<input />
```

- [ ] Há loading states?
- [ ] Há feedback de erro?
- [ ] Há confirmação para ações destrutivas?
- [ ] Textos são claros? (não genéricos)

---

## Como Fazer um Bom Comentário

### Comentários Construtivos

**❌ Ruim:**
```
"Isso está errado"
"Por que você fez assim?"
"Ninguém faz desse jeito"
```

**✅ Bom:**
```
"Acho que aqui seria melhor usar useCallback para evitar re-renders desnecessários. Você concorda?"

"Segundo o DOMAIN_SERVICE_PATTERN.md, mappers devem ficar em app/service/core/[modulo]/mappers/. Podia mover este arquivo?"

"Esse número 50 aparece em vários lugares. Que tal criar uma constante TABLE_PAGE_SIZE no topo do arquivo?"
```

### Estrutura de um Bom Comentário

1. **Contexto:** "Segundo o padrão X..."
2. **Observação:** "Notei que..."
3. **Sugestão:** "Que tal...?"
4. **Por quê:** "...porque melhora a manutenibilidade"

**Template:**
```markdown
**Sugestão:** Mover este componente para `presentation/components/`

**Contexto:** Segundo nossa arquitetura (DOMAIN_SERVICE_PATTERN.md), 
componentes devem ficar na pasta `presentation/components/`, não na raiz do domain.

**Código sugerido:**
```typescript
// Antes
import { UserForm } from '@domain/users/UserForm';

// Depois
import { UserForm } from '@domain/users/presentation/components';
```

**Por quê:** Mantém a arquitetura consistente e facilita descobrir onde os 
componentes estão quando outro dev trabalhar neste módulo.
```

---

### Quando Ser Mais Rígido

**Rejeitar mudanças para:**
- Violações de SOLID + DDD
- Quebra do DOMAIN_SERVICE_PATTERN.md
- Texto hardcoded (sem useTranslation)
- Magic numbers sem constantes
- PR muito grande (> 15 arquivos)
- Testes quebrados
- Breaking changes sem discussão

**Ser mais flexível em:**
- Nomes de variáveis (opinião pessoal)
- Estrutura exata de comentários
- Pequenos magic numbers óbvios (e.g., `Math.max(n, 1)`)
- Espaçamento e formatação (Prettier resolve)

---

## Quando Rejeitar ou Solicitar Mudanças

### ❌ Rejeitar (Request Changes)

Quando há um desses problemas:

1. **Viola padrão arquitetural do projeto**
   ```
   "Esta service não segue o padrão DOMAIN_SERVICE_PATTERN.md. 
   Precisa ter separação entre DTOs, Mappers, Services e Interfaces."
   ```

2. **Texto hardcoded sem useTranslation**
   ```
   "Este texto 'Usuário inativo' está hardcoded. 
   Use useTranslation() e adicione chave em translations.d.ts e pt.br.ts"
   ```

3. **Magic numbers**
   ```
   "Por favor, crie constante para o número 1000 que aparece aqui.
   Criar: const PAGE_SIZE_LIMIT = 1000;"
   ```

4. **Prop-drilling excessivo**
   ```
   "Este componente está passando 8 props diferentes. 
   Considere usar Context API ou re-estruturar o componente."
   ```

5. **PR muito grande**
   ```
   "Esta PR altera 23 arquivos. É difícil revisar mudanças tão grandes.
   Pode dividir em PRs menores? (e.g., domain/schema, service/mappers, etc)"
   ```

6. **Testes quebrados**
   ```
   "Os testes existentes falharam. Precisa corrigir antes de mergear."
   ```

7. **SOLID violations óbvias**
   ```
   "Esta classe tem 15 responsabilidades diferentes. 
   Violarfício o Single Responsibility. Pode refatorar separando em classes menores?"
   ```

---

### ⚠️ Comentar (Comment)

Quando há sugestão, não problema crítico:

```markdown
**Sugestão:** Aqui poderíamos usar useCallback para evitar re-renders.

Não é crítico para esta PR, mas se notar performance issues depois, 
considere adicionar:

```typescript
const handleUpdate = useCallback((user) => {
    updateUser(user);
}, [updateUser]);
```
```

---

### ✅ Aprovar (Approve)

Quando:
- ✅ Segue padrões do projeto
- ✅ Código é limpo e testável
- ✅ Sem magic numbers
- ✅ Tradução correta
- ✅ Não há prop-drilling excessivo
- ✅ React best practices
- ✅ Tamanho razoável

---

## Checklist Final Resumido

Use este checklist ao revisar uma PR:

### 1. Escopo
- [ ] PR resolve apenas um problema/feature?
- [ ] Número de arquivos é razoável (< 15)?
- [ ] Não há commits não relacionados?

### 2. Arquitetura
- [ ] Domain segue `DOMAIN_SERVICE_PATTERN.md`?
- [ ] Service segue `DOMAIN_SERVICE_PATTERN.md`?
- [ ] Estrutura de pastas é consistente?
- [ ] Não há duplicação de código?

### 3. Código
- [ ] Nenhum texto hardcoded? (todos com `useTranslation`)
- [ ] Sem magic numbers? (tudo em constantes)
- [ ] Sem prop-drilling excessivo? (use Context se > 3-5 props)
- [ ] Lógica em hooks, não em componentes?

### 4. SOLID
- [ ] SRP: cada arquivo tem uma responsabilidade?
- [ ] OCP: novo tipo requer modificação de código existente?
- [ ] LSP: interfaces refletem comportamentos reais?
- [ ] ISP: props não têm muitas coisas desnecessárias?
- [ ] DIP: há acoplamento direto a libs?

### 5. React
- [ ] memo() usado apropriadamente?
- [ ] lazy() usado para componentes pesados?
- [ ] useCallback/useMemo em lugares certos?
- [ ] Dependencies arrays completos?

### 6. Performance
- [ ] Bundle size aumentou? (validar)
- [ ] Há renders desnecessários?
- [ ] Queries retornam muitos dados?

### 7. Testes & A11y
- [ ] Testes passam?
- [ ] Sem breaking changes?
- [ ] Acessível? (labels, aria-labels, semântica)

---

## Dúvidas Frequentes

### "Quanto de prop-drilling é demais?"

> Regra: Se um componente está repassando props que **não usa**, e faz isso em > 2 níveis, considere Context.

### "Devo usar useMemo em tudo?"

> Não. Use apenas quando:
> 1. Cálculo é pesado (filtro/sort de grande lista)
> 2. Objeto é passado para componente memo() 
> 3. Notar re-renders desnecessários (medir com Profiler)

### "Quando usar useReducer vs useState?"

> Use useReducer quando tiver > 3 states relacionados ou lógica complexa de update.

### "É OK fazer PR grande se for refactor?"

> Não. Refactors devem ser pequenos e graduais. Se quer refatorar muita coisa:
> 1. Avise a equipe primeiro
> 2. Divida em PRs pequenas
> 3. Cada PR é de uma "camada" ou "módulo"

### "E se o padrão DOMAIN_SERVICE_PATTERN.md não se aplica aqui?"

> Levante com a equipe. Pode ser que:
> 1. É caso especial (utils, helpers)
> 2. Padrão precisa ser atualizado
> 3. Nova abordagem é melhor

---

## Referências

- **DOMAIN_SERVICE_PATTERN.md** - Padrão arquitetural do projeto
- **SOLID Principles** - Uncle Bob (Robert C. Martin)
- **React Best Practices** - React Documentation
- **i18n** - react-i18next documentation
- **Code Review Best Practices** - Google Engineering

---

**Última atualização:** Fevereiro 2026  
**Mantido por:** Equipe de Desenvolvimento - Pontua
