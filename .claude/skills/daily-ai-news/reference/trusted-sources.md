# Trusted sources (allow-list)

The `daily-ai-news` skill may only cite stories from publishers on this list. The URL of each selected story must be fetched and verified with `WebFetch` before it appears in the final article.

If a story deserves coverage but its publisher is not on this list, **do not include it**. Propose the addition to the maintainer instead.

## Thai-language sources

- **Blognone** — https://www.blognone.com/
- **NECTEC (สวทช.)** — https://www.nectec.or.th/news/
- **depa (สำนักงานส่งเสริมเศรษฐกิจดิจิทัล)** — https://www.depa.or.th/th/article-view

## Notes

- Prefer the original publisher over an aggregator. If Reuters and a syndicator both carry the same story, cite Reuters.
- Primary announcements (vendor blogs, arXiv papers) beat commentary when both exist.
- A "trusted" source can still be wrong — summarise what they reported, not what is true in the world.
