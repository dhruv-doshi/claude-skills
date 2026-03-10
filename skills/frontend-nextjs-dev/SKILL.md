---
name: frontend-nextjs-dev
description: Builds frontend next js applications that are ready to be deployed with only minor set-up. Handles all frontend operations like routing, authentication, state management, UI components, API integrations, responsive design, SEO optimization, and testing setups. Generates production-ready code using Next.js 14+ with App Router, TypeScript, Tailwind CSS, and Shadcn/UI components by default, ensuring scalability and best practices for modern web apps. Supports customizations like authentication, databases, and deployment configs (Vercel/Netlify). Outputs a complete project structure with README, .env.example, and deployment scripts. Use always when user is working on "frontend directory", "frontend dev", "frontend code", "building the frontend", "set up the frontend", "build a nextjs", "edit the nextjs" or "nextjs".
version: 1.0.0
---

# Next.js Development

This skill guides creation of distinctive, production-grade frontend interfaces that avoid generic "AI slop" aesthetics. Implement real working code with exceptional attention to aesthetic details and creative choices.

The user provides frontend requirements: a component, page, application, or interface to build. They may include context about the purpose, audience, or technical constraints.

## Design Thinking

Before coding, understand the context and commit to a BOLD aesthetic direction:
- **Purpose**: What problem does this interface solve? Who uses it?
- **Tone**: Pick an extreme: brutally minimal, maximalist chaos, retro-futuristic, organic/natural, luxury/refined, playful/toy-like, editorial/magazine, brutalist/raw, art deco/geometric, soft/pastel, industrial/utilitarian, etc. There are so many flavors to choose from. Use these for inspiration but design one that is true to the aesthetic direction.
- **Constraints**: Technical requirements (framework, performance, accessibility).
- **Differentiation**: What makes this UNFORGETTABLE? What's the one thing someone will remember?

**CRITICAL**: Choose a clear conceptual direction and execute it with precision. Bold maximalism and refined minimalism both work - the key is intentionality, not intensity.

Then implement working code (HTML/CSS/JS, React, Vue, etc.) that is:
- Production-grade and functional
- Visually striking and memorable
- Cohesive with a clear aesthetic point-of-view
- Meticulously refined in every detail

## Best Practices

- **Never put secrets in Client Components** — keep API keys and DB calls server-side
- **Use Server Actions for mutations**, not API routes (unless building a public API)
- **Return errors from Server Actions** instead of throwing (better UX for forms)
- **Use `React.cache()`** to deduplicate DB calls across a single server render
- **Avoid barrel files** (`index.ts` re-exporting everything) — they break tree-shaking
- **Use `revalidateTag`** over `revalidatePath` for precise cache invalidation
- **Avoid runtime CSS-in-JS** (emotion, styled-components) — incompatible with RSC

## Important Points
NEVER use generic AI-generated aesthetics like overused font families (Inter, Roboto, Arial, system fonts), cliched color schemes (particularly purple gradients on white backgrounds), predictable layouts and component patterns, and cookie-cutter design that lacks context-specific character.

Interpret creatively and make unexpected choices that feel genuinely designed for the context. No design should be the same. Always give option to switch between light and dark themes. Always use different fonts, different aesthetics. NEVER converge on common choices (Space Grotesk, for example) across generations.

Always ensure the web application that is being developed is responsive, that is it can be used both on mobile devices as well as laptop or PCs. The layout can be slightly different, for example you can use hamburger icon on mobile screens but use a horizontal navbar for bigger screens. Use the best practises for responsive design. 

**IMPORTANT**: Match implementation complexity to the aesthetic vision. Maximalist designs need elaborate code with extensive animations and effects. Minimalist or refined designs need restraint, precision, and careful attention to spacing, typography, and subtle details. Elegance comes from executing the vision well.

Remember: Claude is capable of extraordinary creative work. Don't hold back, show what can truly be created when thinking outside the box and committing fully to a distinctive vision.
