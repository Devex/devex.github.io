# Devex Tech Blog

```ruby
Devex.do(:good).well()
```

## How to write a Blog post

- Fork this repository
- Branch from `source`, eg. `git checkout -b new-awesome-post`
- Run the Rake task to generate a new post (`rake new_post`)
- Provide a title in clear text
- Find your newly created post in `source/_posts/` and edit it
- You can add an `author` tag to the metadata at top (GitHub username)
- If you don't provide an author, it will be displayed as "Devex", which is fine
  for generic posts like friday links...
- Add at least one category (see below)
- Commit your changes and create a PR against the `source` branch on
  `Devex/devex.github.io`
- Once your PR gets the required amount of `+1`s it gets merged which triggers
  the deployment of your changes

## Categories

For consistency please make sure to only use these categories:

- `Friday Links`: For posts containing our weekly-ish friday links
- `Code`: Anything related to writing code
- `Infra`: For posts about infra and devops matters
- `Planning`: For posts about planning stuff like how we do agile
- `Insight`: For behind-the-scenes posts that talk about how certain features on
  devex.com work
- `Other`: For posts that don't fit into any of the above

Feel free to propose new categories if you see the need!

## Octopress Documentation

Check out [Octopress.org](http://octopress.org/docs) for guides and documentation.
It should all apply to our current stable version (found in the `master`
branch). If this is not the case, [please submit a
fix to our docs repo](https://github.com/octopress/docs).
