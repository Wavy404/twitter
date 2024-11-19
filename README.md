# twitter
pip install tweepy telethon requests beautifulsoup4
import tweepy
from telethon import TelegramClient, events
import re
import requests
from bs4 import BeautifulSoup

# Twitter API credentials (replace with your own keys)
twitter_api_key = 'YOUR_TWITTER_API_KEY'
twitter_api_secret = 'YOUR_TWITTER_API_SECRET'
twitter_access_token = 'YOUR_TWITTER_ACCESS_TOKEN'
twitter_access_secret = 'YOUR_TWITTER_ACCESS_SECRET'

# Set up Twitter API client
auth = tweepy.OAuthHandler(twitter_api_key, twitter_api_secret)
auth.set_access_token(twitter_access_token, twitter_access_secret)
twitter_api = tweepy.API(auth)

# Telegram API credentials (replace with your own)
telegram_api_id = 'YOUR_TELEGRAM_API_ID'
telegram_api_hash = 'YOUR_TELEGRAM_API_HASH'
telegram_client = TelegramClient('session_name', telegram_api_id, telegram_api_hash)

# Define target accounts and patterns for tickers, contract addresses
twitter_accounts = ['CryptoNobler', 'Danny_Crypton', 'DefiWimar']
ticker_pattern = re.compile(r'\$\w+')
contract_pattern = re.compile(r'0x[a-fA-F0-9]{40}')

# Function to fetch tweets from specified accounts
def fetch_twitter_mentions():
    results = []
    for account in twitter_accounts:
        try:
            tweets = twitter_api.user_timeline(screen_name=account, count=100, tweet_mode='extended')
            for tweet in tweets:
                tickers = ticker_pattern.findall(tweet.full_text)
                contracts = contract_pattern.findall(tweet.full_text)
                if tickers or contracts:
                    results.append({
                        'account': account,
                        'tweet': tweet.full_text,
                        'tickers': tickers,
                        'contracts': contracts
                    })
        except tweepy.TweepError as e:
            print(f"Error fetching tweets for {account}: {e}")
    return results

# Function to check project's data on TweetScout
def analyze_project_via_tweetscout(project_handle):
    url = f"https://x.com/TweetScout_io/{project_handle}"
    response = requests.get(url)
    if response.status_code == 200:
        soup = BeautifulSoup(response.text, 'html.parser')
        overall_score = soup.find('span', {'id': 'overall-score'}).text if soup.find('span', {'id': 'overall-score'}) else 'N/A'
        known_followers = soup.find('span', {'id': 'known-followers'}).text if soup.find('span', {'id': 'known-followers'}) else 'N/A'
        trust_level = soup.find('span', {'id': 'trust-level'}).text if soup.find('span', {'id': 'trust-level'}) else 'N/A'
        return overall_score, known_followers, trust_level
    return 'N/A', 'N/A', 'N/A'

# Function to handle new Telegram messages
@telegram_client.on(events.NewMessage)
async def telegram_message_handler(event):
    message = event.message.message
    tickers = ticker_pattern.findall(message)
    contracts = contract_pattern.findall(message)
    if tickers or contracts:
        print(f"Detected tickers in Telegram message: {tickers}\nContracts: {contracts}")
        for contract in contracts:
            # Optional: Use contract for additional analysis (e.g., DEX Screener/Pump.fun checks)

# Main function to run the bot
def main():
    print("Fetching Twitter mentions...")
    twitter_results = fetch_twitter_mentions()
    for result in twitter_results:
        print(f"Account: {result['account']}")
        print(f"Tweet: {result['tweet']}")
        print(f"Tickers: {', '.join(result['tickers'])}")
        print(f"Contracts: {', '.join(result['contracts'])}\n")

        # Example usage of TweetScout analysis (if project handle is available)
        project_handle = result['account']  # Use a more appropriate handle if needed
        overall_score, known_followers, trust_level = analyze_project_via_tweetscout(project_handle)
        print(f"TweetScout Analysis for {project_handle}:")
        print(f"Overall Score: {overall_score}")
        print(f"Known Followers: {known_followers}")
        print(f"Trust Level: {trust_level}\n")

    print("Listening to Telegram channels...")
    with telegram_client:
        telegram_client.run_until_disconnected()

if __name__ == '__main__':
    main()
