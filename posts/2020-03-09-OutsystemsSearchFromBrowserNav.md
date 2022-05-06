# Search Outsystems from Browser's navbar

I thought I'd share a tiny trick for those starting out with Outsystems and coming from other techs, where searching for issues/docs was an exercise of ctrl+t+stackoverflow+tab(or simillar) and search away. I actually mentioned this on a usability session with Outsystems where I found myself kind of lost between bookmarks and search engines trying to look up something.

On to it (*the following is suitable for chromium engines, but probably similar on others):

- Open your browser settings page (usually under the ellipsis buttons on the top right)
- Select the "Search Engine" tab on the left pane, then click on "Manage Search Engines"
- Click Add
- Fill in the following:
Search Engine: Outsystems
Keyword: outsystems.com (or just outsystems)
URL: <https://www.outsystems.com/Community/st_search.aspx?q=%s>

That's it!

Now whenever you hit your navbar, writing outsystems.com followed by a tab or space will redirect your search to the global community search where you can further filter by forums, forge, ideas, etc.

Happy searching!
