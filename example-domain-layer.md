File `services/servicebsvc/internal/domain/user/user.go`

```go
package user

import "time"

// User is an aggregate root representing a user account.
type User struct {
    id             string
    email          Email
    hashedPassword HashedPassword
    createdAt      time.Time
    updatedAt      time.Time
}

// NewUser creates a new user with invariants validated.
func NewUser(id string, email Email, hashedPassword HashedPassword) *User {
    return &User{
        id:             id,
        email:          email,
        hashedPassword: hashedPassword,
        createdAt:      time.Now(),
        updatedAt:      time.Now(),
    }
}

// ChangePassword validates and changes the user's password.
func (u *User) ChangePassword(oldPassword string, newPassword string) error {
    // Verify old password
    if !u.hashedPassword.Verify(oldPassword) {
        return ErrInvalidPassword
    }

    // Validate new password strength
    if err := ValidatePasswordStrength(newPassword); err != nil {
        return err
    }

    // Hash and store new password
    newHash, err := HashPassword(newPassword)
    if err != nil {
        return err
    }

    u.hashedPassword = newHash
    u.updatedAt = time.Now()
    return nil
}

// VerifyPassword checks if the given password matches.
func (u *User) VerifyPassword(password string) bool {
    return u.hashedPassword.Verify(password)
}

// Getters
func (u *User) ID() string { return u.id }
func (u *User) Email() Email { return u.email }
func (u *User) CreatedAt() time.Time { return u.createdAt }
func (u *User) UpdatedAt() time.Time { return u.updatedAt }
```

File `/servicebsvc/internal/domain/user/email.go`:

```go
package user

import (
    "errors"
    "regexp"
    "strings"
)

// Email is a value object representing a valid email address.
type Email struct {
    value string
}

var emailRegex = regexp.MustCompile(`^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`)

// NewEmail creates an Email value object with validation.
func NewEmail(email string) (Email, error) {
    email = strings.TrimSpace(strings.ToLower(email))

    if email == "" {
        return Email{}, errors.New("email cannot be empty")
    }

    if !emailRegex.MatchString(email) {
        return Email{}, errors.New("invalid email format")
    }

    return Email{value: email}, nil
}

func (e Email) String() string {
    return e.value
}

func (e Email) Equals(other Email) bool {
    return e.value == other.value
}
```

File `services/servicebsvc/internal/domain/user/repository.go`

```go
package user

import "context"

// Repository defines persistence operations for users.
// This interface is owned by the domain layer.
// It will be implemented by an adapter (e.g., PostgresUserRepository).
type Repository interface {
    Save(ctx context.Context, user *User) error
    FindByID(ctx context.Context, id string) (*User, error)
    FindByEmail(ctx context.Context, email Email) (*User, error)
    Delete(ctx context.Context, id string) error
}
```
