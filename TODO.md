Not worth using issues just for myself

* Start tagging commits as `[Content]`, `[Jekyll]`, or `[Tooling]`
* Content
  * Go back and see what stuff I just discussed in Discord chats or anonymous internet comments, to see what could have been a post
  * `#include private notes` - Not putting everything here :)
  * Header images. Use Creative Commons or own photos. Use a preprocesser to generate low and high res versions
  * Dark versions of charts in `binpacking`
* Site
  * Actually make use of tags
  * Previous/next links at the bottom of the page
  * Try rebasing onto a new Jekyll theme. Or writing my own in full
  * Consider Hugo but probably not. **Maintain old links**
  * Rewrite the whole thing in XSLT for lols (I had that idea since before everyone started talking about its deprecation 😂)
* Setup
  * Use either Nix, Distrobox, or Docker, to remove dependency on system Ruby
    * Extra: Ruby deps in Nix
    * Extra: GH build using Nix (with caching)
  * Use Just or similar for task running
    * Precondition check that we're in a Nix env
  * Automated tests. Could be as simple as basic checks of the HTML output, up to full browser automation against the live deployment
    * Hard part will be making these not be most of the repo lol
    * Staging deploy? Look up Github docs on this
  * Autoformat (or at least lint) for owt?