# bridge-realdata

بازتولید حمله‌های پل روی داده و state واقعی زنجیره (فورک mainnet)، به‌عنوان مکمل
testbed مصنوعی پایان‌نامه. این پروژه جداست تا اعداد فصل ۵ مخزن اصلی دست‌نخورده بماند
و قراردادهای واقعی (که با solc های قدیمی‌تر کامپایل می‌شوند) تداخلی ایجاد نکنند.

این بسته سه چیز را فراهم می‌کند:
1. ساخت دیتاست واقعی حوادث (فاز A) از یک منبع عمومی قابل بازتولید.
2. بازتولید حمله‌ی واقعی Nomad روی state واقعی mainnet، به‌همراه اصلاح ریشه و تست «پیش/پس».
3. هارنس اندازه‌گیری گس روی تراکنش‌های مشروع واقعی (حل مشکل «انحراف معیار صفر»).

> توجه مهم: تست‌های داخل `test/fork/` به یک RPC آرشیوی mainnet نیاز دارند و بدون
> آن اجرا نمی‌شوند. کامپایل (`forge build`) بدون RPC کار می‌کند، اجرای تست‌ها نه.

---

## پیش‌نیازها

- Foundry (نسخه‌ی تأییدشده‌ی این بسته: `1.5.1`). نصب: https://book.getfoundry.sh
- یک RPC آرشیوی mainnet. Alchemy پلن رایگان کافی است (از ایران معمولاً VPN لازم است).
- کلید API اتراسکن (رایگان) برای جمع‌آوری تراکنش‌های مشروع.
- Python 3 برای اسکریپت‌های دیتاست و تحلیل.

---

## راه‌اندازی

```bash
cp .env.example .env
# مقادیر MAINNET_RPC_URL و ETHERSCAN_API_KEY را در .env بگذارید

forge build          # باید بدون خطا کامپایل شود (بدون نیاز به RPC)
```

برای اینکه Foundry متغیرهای `.env` را ببیند، یا از `--rpc-url $MAINNET_RPC_URL` استفاده
کنید یا قبل از اجرا متغیرها را لود کنید:

```bash
# لینوکس / مک
export $(grep -v '^#' .env | xargs)
# ویندوز PowerShell
Get-Content .env | Where-Object {$_ -notmatch '^#'} | ForEach-Object { $p=$_.Split('=',2); [Environment]::SetEnvironmentVariable($p[0],$p[1]) }
```

---

## گام ۱: دیتاست واقعی حوادث (فاز A)

```bash
python3 scripts/fetch_incidents.py
```
خروجی `data/incidents_raw.csv` است (حوادث پل در بازه‌ی مارس ۲۰۲۱ تا دسامبر ۲۰۲۳،
از API عمومی DefiLlama، معیار IC4). سپس ستون‌های `subtype`، `stride`، `ccf_component`،
`has_poc` و `include_decision` را دستی و طبق معیارهای IC1 تا IC3 فصل ۳ تکمیل کنید.
همین فیلتر قابل‌بازتولید، جمله‌ی «۱۲ حادثه از REKT» را به یک دیتاست عمومی تبدیل می‌کند.

---

## گام ۲: حمله‌ی واقعی Nomad + تست پیش/پس

ابتدا slot نگاشت `confirmAt` را روی قرارداد واقعی به‌صورت تجربی تأیید کنید:

```bash
forge test --match-test test_FindConfirmAtSlot -vvv
```
عددی که چاپ می‌شود (slot ای که برای کلید `0x00` مقدار `1` دارد) را در بالای فایل
`test/fork/NomadReal.t.sol` در ثابت `CONFIRM_AT_SLOT` بگذارید.

سپس:

```bash
# مرحله ۱: حمله روی قرارداد واقعی در بلوک قبل از هک، باید موفق شود
forge test --match-test test_RealNomad_AttackSucceeds -vvv

# مرحله ۳: همان حمله پس از اصلاح state، باید شکست بخورد
forge test --match-test test_RealNomad_AttackFails_AfterStateFix -vvv
```

این جفت، همان پروتکل «پیش/پس» پایان‌نامه است، این بار روی state واقعی mainnet.
اصلاح به‌کاررفته دقیقاً ریشه‌ی واقعی حادثه است: برگرداندن `confirmAt[0x00]` از `1`
به `0`. پیش از نوشتن، تست تأیید می‌کند مقدار فعلی واقعاً `1` است (تا داور مطمئن شود
slot درست انتخاب شده).

اجرای اول کند است چون state از RPC دانلود می‌شود؛ بعد در `~/.foundry/cache/rpc`
کش می‌شود.

---

## گام ۳: گس روی تراکنش‌های مشروع واقعی

```bash
export $(grep -v '^#' .env | xargs)
python3 scripts/fetch_legit_txs.py           # data/legit_txs.json
forge test --match-test test_ReplayLegitTxs_MeasureGas -vv   # data/gas_real.csv
python3 scripts/analyze_gas.py               # میانگین/انحراف معیار/میانه
```
برای هر تراکنش، state دقیقاً قبل از همان تراکنش فورک و همان فراخوانی دوباره اجرا
می‌شود. چون calldata تراکنش‌های واقعی با هم فرق دارد، توزیع واقعی گس به دست می‌آید و
مشکل «انحراف معیار صفر» رفع می‌شود.

نکته: اگر اسکریپت تراکنشی پیدا نکرد، `WANTED_SELECTORS` در `scripts/fetch_legit_txs.py`
را خالی کنید یا selector ها را با `cast sig 'process(bytes)'` تأیید کنید.

---

## گام ۴ (اختیاری): Chainswap و Poly Network

قالب `test/fork/ChainswapReal.t.sol` آماده است و فعلاً `skip` می‌شود. برای تکمیل،
مراحل داخل همان فایل را دنبال کنید (گرفتن سورس proxy با `forge clone`، استخراج
selector و امضاها از PoC، و برداشتن `vm.skip`). ریشه‌ی واقعی Chainswap با مدل فعلی
testbed کمی فرق دارد؛ توضیحش در همان فایل آمده و باید در فصل ۲ هم اصلاح شود.

---

## اجرای همه‌ی تست‌های فورک

```bash
export $(grep -v '^#' .env | xargs)
forge test --match-path "test/fork/*" -vv
```

---

## نقشه‌ی فایل‌ها

```
foundry.toml                      پیکربندی (rpc از .env)
.env.example                      الگوی متغیرهای محیطی
scripts/fetch_incidents.py        فاز A: دیتاست حوادث از DefiLlama
scripts/fetch_legit_txs.py        تراکنش‌های مشروع واقعی از Etherscan
scripts/analyze_gas.py            تحلیل آماری گس
test/fork/NomadReal.t.sol         حمله‌ی واقعی Nomad + یافتن slot + پیش/پس
test/fork/GasReplayReal.t.sol     گس روی تراکنش‌های مشروع واقعی
test/fork/ChainswapReal.t.sol     قالب Chainswap (skip تا تکمیل)
data/                             خروجی اسکریپت‌ها (در .gitignore)
```

---

## منابع

- الگو و calldata حمله‌ها: DeFiHackLabs — https://github.com/SunWeb3Sec/DeFiHackLabs
- دیتاست حوادث: DefiLlama Hacks API — https://api.llama.fi/hacks
- تحلیل ریشه‌ی Nomad: CertiK post-mortem و تحلیل samczsun
- چارچوب دسته‌بندی: Notland et al., "SoK: Cross-Chain Bridging..." (arXiv:2403.00405)

---

## محدودیت‌ها (صداقت منبع)

- محیط توسعه‌ی نویسنده‌ی این بسته به RPC بلاکچین دسترسی نداشت، پس تست‌های فورک فقط
  کامپایل تأیید شده‌اند و باید روی سیستم شما با RPC واقعی اجرا شوند.
- Wormhole (سولانا) و BNB Chain (اثبات IAVL) با فورک EVM بازتولید نمی‌شوند؛ خارج از دامنه.
- Polygon Plasma هیچ‌وقت exploit نشد (با bug bounty پیدا شد)؛ بازتولیدش proof واقعی
  لازم دارد. Gnosis Omni روی زنجیره‌ی ETHPoW بود و RPC آرشیوی‌اش کمیاب است.
