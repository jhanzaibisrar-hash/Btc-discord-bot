import os
import asyncio
import requests
import discord
from discord.ext import commands

TOKEN = os.getenv("DISCORD_TOKEN")

intents = discord.Intents.default()
bot = commands.Bot(command_prefix="!", intents=intents)

running = False
task = None


def get_klines(interval, limit=100):
    url = "https://api.binance.com/api/v3/klines"
    params = {
        "symbol": "BTCUSDT",
        "interval": interval,
        "limit": limit
    }
    r = requests.get(url, params=params, timeout=10)
    r.raise_for_status()
    return r.json()


def analyze_btc():
    # 5-minute candles
    candles = get_klines("5m", 100)

    # Ignore the currently forming candle
    closed = candles[:-1]

    last = closed[-1]
    prev = closed[-2]

    last_open = float(last[1])
    last_close = float(last[4])
    prev_open = float(prev[1])
    prev_close = float(prev[4])

    # Recent momentum
    last_move = (last_close - last_open) / last_open * 100
    prev_move = (prev_close - prev_open) / prev_open * 100

    # 15-minute trend using 15m candles
    candles15 = get_klines("15m", 50)
    closed15 = candles15[:-1]

    closes = [float(x[4]) for x in closed15]

    ema_fast = sum(closes[-5:]) / 5
    ema_slow = sum(closes[-20:]) / 20

    trend_up = ema_fast > ema_slow
    trend_down = ema_fast < ema_slow

    score = 0

    if last_close > last_open:
        score += 1
    else:
        score -= 1

    if prev_close > prev_open:
        score += 1
    else:
        score -= 1

    if trend_up:
        score += 1
    elif trend_down:
        score -= 1

    if score >= 2:
        signal = "🟢 UP"
        reason = "Short-term momentum + 15m trend bullish"
    elif score <= -2:
        signal = "🔴 DOWN"
        reason = "Short-term momentum + 15m trend bearish"
    else:
        signal = "⚪ NO TRADE"
        reason = "Momentum mixed / setup weak"

    return signal, reason, last_close


async def analysis_loop(channel):
    global running

    while running:
        try:
            signal, reason, price = analyze_btc()

            message = (
                f"**BTC 10-Minute Signal**\n\n"
                f"**Signal:** {signal}\n"
                f"**BTC:** ${price:,.2f}\n"
                f"**Reason:** {reason}\n\n"
                f"Next check: 10 minutes"
            )

            await channel.send(message)

        except Exception as e:
            await channel.send(f"⚠️ Analysis error: `{e}`")

        await asyncio.sleep(600)


@bot.event
async def on_ready():
    try:
        await bot.tree.sync()
        print(f"Logged in as {bot.user}")
    except Exception as e:
        print("Command sync error:", e)


@bot.tree.command(name="analyze", description="Start BTC 10-minute signal analysis")
async def analyze(interaction: discord.Interaction):
    global running, task

    if running:
        await interaction.response.send_message(
            "🟡 Analysis already running."
        )
        return

    running = True

    await interaction.response.send_message(
        "🟢 BTC analysis started. Signal har 10 minutes mein check hoga."
    )

    task = asyncio.create_task(analysis_loop(interaction.channel))


@bot.tree.command(name="stop", description="Stop BTC signal analysis")
async def stop(interaction: discord.Interaction):
    global running, task

    running = False

    if task:
        task.cancel()
        task = None

    await interaction.response.send_message(
        "🛑 BTC analysis stopped."
    )


if not TOKEN:
    raise RuntimeError("DISCORD_TOKEN environment variable missing")

bot.run(TOKEN)
