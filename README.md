# Matryoshka

A multi-board task tracker that works like a Russian doll — each board nested inside a single workspace view, with a mood-board home that shows all your lists at once.

Built by Potchkey Labs.

## Features

- Multiple boards, each with multiple lists
- Parent/child task hierarchy with collapse arrows
- Custom columns (text, status, who, due date, chips)
- Templates: Blank, Corporate Tie-Out, Financing Matter
- Dashboard home with draggable board tiles
- Art-of-the-day background rotation
- localStorage persistence (Supabase cloud sync coming)

## Roadmap

- [ ] Google OAuth via Supabase
- [ ] Per-user board isolation with RLS
- [ ] Board collaborator invites
- [ ] Matryoshka home view (mood-board of all boards)
- [ ] Vercel deploy

## Development

Open `index.html` over a local server (not directly as a `file://`):

```bash
python3 -m http.server 8080
# then open http://localhost:8080
```

## License

MIT
