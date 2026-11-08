# Project README Import

## Decision

Project README files are imported as PersonaOS-owned Markdown snapshots. Public project pages do not fetch or render live repository README files.

The import pipeline uses the configured GitHub token only on the backend to read repository content. Imported projects are saved as drafts so a private repository's README or images are never published without an explicit dashboard publish action.

## Import Pipeline

```text
Dashboard repository URL
  -> backend reads README and referenced images from GitHub
  -> backend formats Markdown and copies image assets to Supabase Storage
  -> backend rewrites Markdown image links to PersonaOS asset routes
  -> backend saves the project Markdown snapshot as a draft
  -> public project page renders the saved Markdown and repository banner
```

The Markdown formatter preserves standard Markdown and GitHub-flavoured structures supported by the existing renderer. It normalizes repository-relative links, fixes ambiguous separators that would create invalid heading layouts, and rejects unsupported image references rather than leaving broken output.

## Project Assets

Imported README images live in a private Supabase Storage bucket named `project-assets`.

- The backend generates paths from the authenticated owner, project slug, and a content hash.
- The importer downloads assets with the stored GitHub token, validates their media type and size, and never exposes that token in generated Markdown or browser responses.
- Dashboard previews use authenticated asset access.
- Public asset routes serve only assets belonging to published projects.
- Publishing a project intentionally makes its imported README images visible with the project. Draft and archived project assets remain inaccessible from public routes.

This avoids public GitHub raw URLs, so assets from private repositories remain available after a project is deliberately published.

## Existing Projects

The backend provides an explicit Docker-run backfill command for existing project documents. It accepts one owner identity, supports `--dry-run`, uses the same formatter and asset import service as new imports, and writes projects as drafts unless the operator explicitly preserves a published status.

The command must use the normal content-save/storage path rather than direct database writes, keeping Markdown objects and `content_documents` metadata consistent.

## Public Project Page

Project pages render their saved portfolio content and a prominent repository banner linking to the configured repository URL. GitHub remains responsible for repository README rendering and access control; PersonaOS does not proxy a live README into the public portfolio.

## Verification

- Unit-test Markdown formatting, relative image rewriting, and unsupported/oversized asset rejection.
- Unit-test asset authorization for drafts, archived projects, and published projects.
- Unit-test the backfill dry-run and persistence paths.
- Browser-test a project with copied screenshots and a private-project draft that produces no public asset access.
