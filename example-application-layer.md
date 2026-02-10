`services/authsvc/internal/application/command/login.go`:

```go
package command

import (
    "context"
    "time"

    "github.com/example/service-manager/services/authsvc/internal/application/ports"
    "github.com/example/service-manager/services/authsvc/internal/domain/session"
    "github.com/example/service-manager/services/authsvc/internal/domain/user"
)

// LoginCommand handles user login.
type LoginCommand struct {
    userRepo      user.Repository
    sessionRepo   session.Repository
    authorClient  ports.AuthorClient
    logger        ports.Logger
}

func NewLoginCommand(
    userRepo user.Repository,
    sessionRepo session.Repository,
    authorClient ports.AuthorClient,
    logger ports.Logger,
) *LoginCommand {
    return &LoginCommand{
        userRepo:     userRepo,
        sessionRepo:  sessionRepo,
        authorClient: authorClient,
        logger:       logger,
    }
}

type LoginInput struct {
    Email    string
    Password string
}

type LoginOutput struct {
    Token      string
    UserID     string
    AuthorName string
    ExpiresAt  time.Time
}

func (c *LoginCommand) Execute(ctx context.Context, input LoginInput) (*LoginOutput, error) {
    // 1. Parse and validate email
    email, err := user.NewEmail(input.Email)
    if err != nil {
        return nil, ErrInvalidCredentials
    }

    // 2. Find user by email
    u, err := c.userRepo.FindByEmail(ctx, email)
    if err != nil {
        if errors.Is(err, user.ErrUserNotFound) {
            return nil, ErrInvalidCredentials
        }
        return nil, err
    }

    // 3. Verify password (domain logic)
    if !u.VerifyPassword(input.Password) {
        c.logger.Warn(ctx, "failed login attempt", ports.LogField{Key: "user_id", Value: u.ID()})
        return nil, ErrInvalidCredentials
    }

    // 4. Create session (domain logic)
    sess := session.New(u.ID(), 24*time.Hour)

    // 5. Enrich with author information (optional, graceful degradation)
    var authorName string
    authorInfo, err := c.authorClient.GetAuthor(ctx, u.ID())
    if err != nil {
        c.logger.Warn(ctx, "failed to fetch author info", ports.LogField{Key: "error", Value: err})
        // Continue without author info
    } else {
        authorName = authorInfo.Name
    }

    // 6. Save session
    if err := c.sessionRepo.Save(ctx, sess); err != nil {
        return nil, err
    }

    // 7. Log success
    c.logger.Info(ctx, "user logged in",
        ports.LogField{Key: "user_id", Value: u.ID()},
        ports.LogField{Key: "session_id", Value: sess.ID()},
    )

    // 8. Return result
    return &LoginOutput{
        Token:      sess.Token().String(),
        UserID:     u.ID(),
        AuthorName: authorName,
        ExpiresAt:  sess.ExpiresAt(),
    }, nil
}
```

`services/authsvc/internal/application/ports/author_client.go`:

```go
package ports

import "context"

// AuthorClient is an outbound port for fetching author information.
// This interface is owned by the application layer.
// It will be implemented by an adapter (e.g., InprocAuthorClient, ConnectAuthorClient).
type AuthorClient interface {
    GetAuthor(ctx context.Context, authorID string) (*AuthorInfo, error)
}

// AuthorInfo is an application DTO for author data.
type AuthorInfo struct {
    ID        string
    Name      string
    Bio       string
    AvatarURL string
}

// Application-level errors
var (
    ErrAuthorNotFound    = errors.New("author not found")
    ErrAuthorServiceDown = errors.New("author service unavailable")
)
```
