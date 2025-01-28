# Cloud-Bot
pip install streamlit requests pyyaml sqlalchemy telebot
import streamlit as st
import requests
import yaml
from sqlalchemy import create_engine, Column, Integer, String, Float, DateTime, Boolean
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker
from datetime import datetime
import telebot

# Load config
def load_config():
    with open("config.yaml", "r") as file:
        return yaml.safe_load(file)

def save_config(config):
    with open("config.yaml", "w") as file:
        yaml.dump(config, file)

config = load_config()

DEXSCREENER_API_URL = config["dex_api_url"]
RUGCHECK_API_URL = config["rugcheck_api_url"]
DATABASE_URL = config["database_url"]
TELEGRAM_BOT_TOKEN = config["telegram_bot_token"]
TELEGRAM_CHAT_ID = config["telegram_chat_id"]
BONKBOT_COMMAND_PREFIX = config["bonkbot_command_prefix"]

# Initialize Telegram bot
telegram_bot = telebot.TeleBot(TELEGRAM_BOT_TOKEN)

# Database setup
Base = declarative_base()

class TokenData(Base):
    __tablename__ = "token_data"
    id = Column(Integer, primary_key=True)
    token_address = Column(String, nullable=False)
    price = Column(Float, nullable=False)
    volume = Column(Float, nullable=False)
    liquidity = Column(Float, nullable=False)
    dev_address = Column(String, nullable=True)  # Developer address
    fake_volume_ratio = Column(Float, nullable=True)  # Fake volume ratio
    is_bundled = Column(Boolean, nullable=False, default=False)  # Bundled supply flag
    timestamp = Column(DateTime, default=datetime.utcnow)

engine = create_engine(DATABASE_URL)
Base.metadata.create_all(engine)

# Fetch token data from DexScreener
def fetch_token_data(token_address):
    """Fetch token data from DexScreener API."""
    response = requests.get(f"{DEXSCREENER_API_URL}{token_address}")
    if response.status_code == 200:
        return response.json()
    else:
        st.error(f"Failed to fetch data for token: {token_address}")
        return None

# Fetch contract analysis from rugcheck.xyz
def fetch_rugcheck_data(token_address):
    """Fetch contract analysis from rugcheck.xyz."""
    params = {"token_address": token_address}
    response = requests.get(RUGCHECK_API_URL, params=params)
    if response.status_code == 200:
        return response.json()
    else:
        st.error(f"Failed to fetch rugcheck data for token: {token_address}")
        return None

# Check if token is blacklisted
def is_blacklisted(token_data):
    """Check if token or dev is blacklisted."""
    token_address = token_data.get("address")
    dev_address = token_data.get("devAddress")

    if token_address in config["blacklist"]["coins"]:
        return True
    if dev_address in config["blacklist"]["devs"]:
        return True
    return False

# Apply filters
def apply_filters(token_data):
    """Apply filters to token data."""
    liquidity = token_data.get("liquidity", 0)
    price_change = token_data.get("priceChange24h", 0)

    if liquidity < config["filters"]["min_liquidity"]:
        return False
    if abs(price_change) > config["filters"]["max_price_change"]:
        return False
    return True

# Check rugcheck.xyz for contract status and bundled supply
def check_rugcheck(token_data):
    """Check if contract is marked as 'Good' and if supply is bundled."""
    rugcheck_data = fetch_rugcheck_data(token_data["address"])
    if rugcheck_data:
        # Check if contract is marked as "Good"
        if rugcheck_data.get("status") != "Good":
            return False

        # Check if supply is bundled
        if rugcheck_data.get("is_bundled", False):
            # Add token and dev to blacklist
            config["blacklist"]["coins"].append(token_data["address"])
            if "devAddress" in token_data:
                config["blacklist"]["devs"].append(token_data["devAddress"])
            save_config(config)
            return False

        return True
    return False

# Save token data
def save_token_data(token_data):
    """Save token data to the database."""
    Session = sessionmaker(bind=engine)
    session = Session()
    token = TokenData(
        token_address=token_data['address'],
        price=token_data['price'],
        volume=token_data['volume'],
        liquidity=token_data['liquidity'],
        dev_address=token_data.get('devAddress'),
        fake_volume_ratio=token_data.get('fake_volume_ratio'),
        is_bundled=token_data.get('is_bundled', False)
    )
    session.add(token)
    session.commit()
    session.close()

# Send Telegram notification
def send_telegram_notification(message):
    """Send a notification via Telegram."""
    telegram_bot.send_message(TELEGRAM_CHAT_ID, message)

# Execute BonkBot trade command
def execute_bonkbot_trade(action, token_address):
    """Send a trade command to BonkBot via Telegram."""
    command = f"{BONKBOT_COMMAND_PREFIX} {action} {token_address}"
    telegram_bot.send_message(TELEGRAM_CHAT_ID, command)
    st.success(f"Sent {action} command for token: {token_address}")

# Streamlit UI
st.title("Crypto Trading Bot")

# Sidebar for configuration
st.sidebar.header("Configuration")
min_liquidity = st.sidebar.number_input("Minimum Liquidity (USD)", value=config["filters"]["min_liquidity"])
max_price_change = st.sidebar.number_input("Maximum Price Change (%)", value=config["filters"]["max_price_change"])
fake_volume_threshold = st.sidebar.number_input("Fake Volume Threshold", value=config["filters"]["fake_volume_threshold"])

# Update config
if st.sidebar.button("Update Filters"):
    config["filters"]["min_liquidity"] = min_liquidity
    config["filters"]["max_price_change"] = max_price_change
    config["filters"]["fake_volume_threshold"] = fake_volume_threshold
    save_config(config)
    st.sidebar.success("Filters updated!")

# Main UI
st.header("Token Monitoring")
token_address = st.text_input("Enter Token Address")

if st.button("Check Token"):
    token_data = fetch_token_data(token_address)
    if token_data and not is_blacklisted(token_data) and apply_filters(token_data) and check_rugcheck(token_data):
        save_token_data(token_data)
        st.success(f"Token {token_address} is valid and saved to the database.")
        execute_bonkbot_trade("buy", token_address)
    else:
        st.error(f"Token {token_address} is invalid or blacklisted.")

# Display blacklists
st.header("Blacklists")
st.subheader("Blacklisted Coins")
st.write(config["blacklist"]["coins"])

st.subheader("Blacklisted Developers")
st.write(config["blacklist"]["devs"])
