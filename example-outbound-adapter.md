`services/authsvc/internal/adapters/outbound/persistence/postgres/user_repository.go`:

```go
package postgres

import (
    "context"
    "database/sql"
    "errors"

    "github.com/example/service-manager/services/authsvc/internal/domain/user"
)

type UserRepository struct {
    db *sql.DB
}

func NewUserRepository(db *sql.DB) *UserRepository {
    return &UserRepository{db: db}
}

func (r *UserRepository) Save(ctx context.Context, u *user.User) error {
    query := `
        INSERT INTO users (id, email, hashed_password, created_at, updated_at)
        VALUES ($1, $2, $3, $4, $5)
        ON CONFLICT (id) DO UPDATE SET
            email = EXCLUDED.email,
            hashed_password = EXCLUDED.hashed_password,
            updated_at = EXCLUDED.updated_at
    `

    _, err := r.db.ExecContext(ctx, query,
        u.ID(),
        u.Email().String(),
        u.HashedPassword().String(), // Assumes getter method
        u.CreatedAt(),
        u.UpdatedAt(),
    )

    return err
}

func (r *UserRepository) FindByEmail(ctx context.Context, email user.Email) (*user.User, error) {
    query := `
        SELECT id, email, hashed_password, created_at, updated_at
        FROM users
        WHERE email = $1
    `

    var (
        id             string
        emailStr       string
        passwordStr    string
        createdAt      time.Time
        updatedAt      time.Time
    )

    err := r.db.QueryRowContext(ctx, query, email.String()).Scan(
        &id, &emailStr, &passwordStr, &createdAt, &updatedAt,
    )

    if errors.Is(err, sql.ErrNoRows) {
        return nil, user.ErrUserNotFound
    }
    if err != nil {
        return nil, err
    }

    // Reconstruct domain object
    e, _ := user.NewEmail(emailStr) // Already validated in DB
    p := user.NewHashedPassword(passwordStr)

    return user.Reconstruct(id, e, p, createdAt, updatedAt), nil
}
```

