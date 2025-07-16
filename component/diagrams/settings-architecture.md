# 設定システムアーキテクチャ図

## 設定システムの全体構造

```mermaid
graph TB
    subgraph "Definition Layer"
        A[ComponentManifest] --> B[SettingsManifest :global]
        A --> C[SettingsManifest :step]
        B --> D[Attribute definitions]
        C --> E[Attribute definitions]
    end
    
    subgraph "Storage Layer"
        F[(decidim_components table)]
        F --> G[settings JSONB column]
        G --> H[global settings]
        G --> I[default_step settings]
        G --> J[steps settings hash]
    end
    
    subgraph "Access Layer"
        K[HasSettings module]
        K --> L[settings method]
        K --> M[current_settings method]
        K --> N[step_settings method]
    end
    
    subgraph "Validation Layer"
        O[Dynamic Schema Class]
        O --> P[Type coercion]
        O --> Q[Validations]
        O --> R[Translations]
    end
    
    D --> O
    E --> O
    F --> K
    O --> L
    O --> M
    O --> N
```

## 設定型とフォームフィールドのマッピング

```mermaid
flowchart LR
    subgraph "Setting Types"
        A[boolean]
        B[integer]
        C[string]
        D[text]
        E[enum]
        F[select]
        G[scope]
        H[array]
        I[float]
        J[time]
    end
    
    subgraph "Form Fields"
        K[check_box]
        L[number_field]
        M[text_field]
        N[text_area/editor]
        O[radio_buttons]
        P[select_field]
        Q[scope_selector]
        R[custom_field]
        S[number_field]
        T[datetime_field]
    end
    
    A --> K
    B --> L
    C --> M
    D --> N
    E --> O
    F --> P
    G --> Q
    H --> R
    I --> S
    J --> T
    
    style A fill:#f96
    style B fill:#f96
    style C fill:#f96
    style D fill:#f96
    style E fill:#69f
    style F fill:#69f
    style G fill:#9f6
    style H fill:#ff6
    style I fill:#f96
    style J fill:#9f6
```

## 条件付き設定の評価フロー

```mermaid
sequenceDiagram
    participant A as Admin UI
    participant F as Form
    participant S as Setting
    participant C as Context
    participant P as Proc/Lambda
    
    A->>F: Render form
    F->>S: Check readonly?
    S->>C: Build context
    Note over C: {component: @component,<br/>current_user: user}
    S->>P: Call readonly proc
    P->>P: Evaluate condition
    P-->>S: true/false
    
    alt Readonly = true
        S-->>F: Disable field
        F-->>A: Show as readonly
    else Readonly = false
        S-->>F: Enable field
        F-->>A: Show as editable
    end
```

## 多言語設定の構造

```mermaid
classDiagram
    class TranslatableAttribute {
        +name: Symbol
        +type: Symbol
        +translations: Hash
        +default_locale: Symbol
    }
    
    class Translation {
        +locale: Symbol
        +value: String
    }
    
    class SettingsJSON {
        announcement: {
            en: "Welcome",
            ja: "ようこそ",
            es: "Bienvenido"
        }
    }
    
    TranslatableAttribute "1" --> "*" Translation
    SettingsJSON --> TranslatableAttribute : stored as
    
    note for TranslatableAttribute "translated: true enables<br/>multi-language support"
```

## 設定の検証パイプライン

```mermaid
flowchart TD
    A[User Input] --> B{Type Check}
    B -->|Valid| C[Type Coercion]
    B -->|Invalid| D[Type Error]
    
    C --> E{Required Check}
    E -->|Present| F[Custom Validations]
    E -->|Missing| G[Presence Error]
    
    F --> H{Inclusion Check}
    H -->|Valid| I[Range Check]
    H -->|Invalid| J[Inclusion Error]
    
    I -->|Valid| K[Save to DB]
    I -->|Invalid| L[Range Error]
    
    D --> M[Return Errors]
    G --> M
    J --> M
    L --> M
    
    K --> N[Success]
    
    style D fill:#f99
    style G fill:#f99
    style J fill:#f99
    style L fill:#f99
    style N fill:#9f9
```