from flask import Flask, Response, request
from feedgen.feed import FeedGenerator
import snscrape.modules.twitter as sntwitter
import html
import os

app = Flask(__name__)

@app.route("/rss")
def rss_feed():
    user = request.args.get("user")
    if not user:
        return "Erreur : ajoute ?user=NomDuCompte à l'URL (ex: /rss?user=ActuFoot_)", 400

    fg = FeedGenerator()
    fg.title(f"Flux RSS Twitter – @{user}")
    fg.link(href=f"https://twitter.com/{user}")
    fg.description(f"Tweets récents de @{user} (avec images)")

    try:
        for i, tweet in enumerate(sntwitter.TwitterUserScraper(user).get_items()):
            print("Tweet trouvé :", tweet)
            if i >= 10:
                break

            link = f"https://twitter.com/{user}/status/{tweet.id}"
            title = tweet.content[:80]
            desc = f"<p>{html.escape(tweet.content)}</p>"

            if tweet.media:
                for media in tweet.media:
                    if hasattr(media, 'fullUrl'):
                        desc += f'<br><img src="{media.fullUrl}" style="max-width:100%;"><br>'

            fg.add_entry()\
              .title(title)\
              .link(href=link)\
              .description(desc)\
              .pubDate(tweet.date)

    except Exception as e:
        return f"Erreur lors de la récupération de @{user} : {str(e)}", 500

    return Response(fg.rss_str(pretty=True), mimetype="application/rss+xml")

# 🚨 Important : ne PAS forcer un port fixe (comme 10000)
app.run(host="0.0.0.0", port=int(os.environ.get("PORT", 5000)))
