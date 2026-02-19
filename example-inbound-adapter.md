# Inbound Adapters

Inbound adapters (primary/driving adapters) deliver requests to the application layer. They translate external requests into application commands/queries.

## Types of Inbound Adapters

Services typically have multiple inbound adapters:
- **Contract adapter** (`internal/adapters/inbound/contracts/`) - For in-process calls from other services
- **HTTP adapter** (`internal/adapters/inbound/http/`) - For REST API endpoints
- **Connect adapter** (`internal/adapters/inbound/connect/`) - For gRPC/Connect calls
- **CLI adapter** (`internal/adapters/inbound/cli/`) - For command-line interfaces

All follow the same pattern: translate external requests → call application layer → translate responses.

---

## Example: HTTP Adapter

`services/servicebsvc/internal/adapters/inbound/http/handlers/login.go`:

```go
package handlers

import (
    "encoding/json"
    "net/http"

    "github.com/example/service-manager/services/servicebsvc/internal/application/command"
)

type LoginHandler struct {
    loginCmd *command.LoginCommand
}

func NewLoginHandler(loginCmd *command.LoginCommand) *LoginHandler {
    return &LoginHandler{loginCmd: loginCmd}
}

type LoginRequest struct {
    Email    string `json:"email"`
    Password string `json:"password"`
}

type LoginResponse struct {
    Token      string `json:"token"`
    UserID     string `json:"user_id"`
    AuthorName string `json:"author_name,omitempty"`
    ExpiresAt  int64  `json:"expires_at"`
}

func (h *LoginHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
    // 1. Parse request
    var req LoginRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "invalid request", http.StatusBadRequest)
        return
    }

    // 2. Call application layer
    input := command.LoginInput{
        Email:    req.Email,
        Password: req.Password,
    }

    output, err := h.loginCmd.Execute(r.Context(), input)
    if err != nil {
        // 3. Map application errors to HTTP status codes
        switch {
        case errors.Is(err, command.ErrInvalidCredentials):
            http.Error(w, "invalid credentials", http.StatusUnauthorized)
        default:
            http.Error(w, "internal server error", http.StatusInternalServerError)
        }
        return
    }

    // 4. Return response
    resp := LoginResponse{
        Token:      output.Token,
        UserID:     output.UserID,
        AuthorName: output.AuthorName,
        ExpiresAt:  output.ExpiresAt.Unix(),
    }

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusOK)
    json.NewEncoder(w).Encode(resp)
}
```


