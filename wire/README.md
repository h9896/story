# 利用 Wire 完成 Dependency Injection

## 為什麼要用依賴注入

Dependency Injection 是為了解決物件高耦合的問題, 來提高程式的可維護性, 並更容易寫出測試程式碼, 下面會有例子來說明

## Install Wire

`go install github.com/google/wire/cmd/wire@latest` and make sure $PATH includes $GOPATH/bin

## Example

有一個 Repo 專門負責資料庫的,有另一個 Repo 想要使用它來建立新的 Service, 這邊需要一個提供資料的provider跟injector(wire.go), 建立完之後 Run `wire`, 就可以看到一個 `wire_gen.go`

---

```go
// database
package example

import (
    "context"
    "database/sql"
    "fmt"
)

type DatabaseConfig struct {
    Dsn string `yaml:"Dsn,omitempty" mapstructure:"Dsn"`
}

func NewDB(ctx context.Context, cfg *DatabaseConfig) (db *sql.DB, err error) {
    if cfg == nil {
        return nil, fmt.Errorf("The DB configuration is empty")
    }
    db, err = sql.Open("postgres", cfg.Dsn)
    if err != nil {
        return nil, err
    }
    if err = db.PingContext(ctx); err != nil {
        return nil, err
    }
    return db, nil
}
```

---

```go
package service
// new service
import (
    "database/sql"
)

type App struct {
    database *sql.DB
}

func (app *App) InitDB(db *sql.DB) {
    app.database = db
}

```

---

```go
package cmd
// provide some wire information
import (
    "context"
    "dd/example"
    "dd/internal/service"

    "github.com/google/wire"
)

var appSet = wire.NewSet(dbConfig, initApp)

func dbConfig() example.DatabaseConfig {
    return example.DatabaseConfig{Dsn: "Dsn"}
}

func initApp(ctx context.Context, cfg example.DatabaseConfig) (*service.App, error) {
    app := &service.App{}
    db, err := example.NewDB(ctx, &cfg)
    if err != nil {
        return nil, err
    }
    app.InitDB(db)
    return app, nil
}
```

---

```go
package cmd
// injector
// wire.go
import (
    "context"
    "dd/internal/service"

    "github.com/google/wire"
)

func initializeApp(ctx context.Context) (*service.App, error) {
    wire.Build(appSet)
    return &service.App{}, nil
}
```

---

```go
// Code generated by Wire. DO NOT EDIT.

//go:generate go run github.com/google/wire/cmd/wire
//go:build !wireinject
// +build !wireinject

package cmd

import (
    "context"
    "dd/internal/service"
)

// Injectors from wire.go:

func initializeApp(ctx context.Context) (*service.App, error) {
    databaseConfig := dbConfig()
    app, err := initApp(ctx, databaseConfig)
    if err != nil {
        return nil, err
    }
    return app, nil
}
```
