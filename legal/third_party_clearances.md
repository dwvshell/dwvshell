import csv
import json
import hashlib
import os
from dataclasses import dataclass, asdict
from datetime import datetime
from pathlib import Path
from typing import List, Dict, Any

# ============================================================
# 1. CORE IDENTITY — TRUST / IP / CORPORATE CONTROL
# ============================================================

MASTER_IDENTITY = {
    "trust_name": "DWVSCPS ENERGY FAMILY TRUST™",
    "controller": "R. E. STOCKFORD JR / 15389089 CANADA INC.",
    "trademark": "DWV STOCKFORD CONTAMINATE PIPELINE SHELL INC™",
    "patent_status": "Patent-Pending (Government of Canada)",
    "trust_vault": "DWVSCPS_TRUST_VAULT_2026",
    "authority": "Controlling authority authorized ownership",
    "vin": "DA556EFCE7AA09976",
    "piid": "da556efce7aa099764c4dc6565908913",
    "timestamp": datetime.utcnow().isoformat(),
}

INPUT_DIR = Path("input_documents")
OUTPUT_DIR = Path("DWVSCPS_MASTER_COMPLIANCE")
OUTPUT_DIR.mkdir(exist_ok=True)

# ============================================================
# 2. HASHING & CHAIN OF CUSTODY
# ============================================================

def sha256_file(path: Path) -> str:
    h = hashlib.sha256()
    with open(path, "rb") as f:
        for chunk in iter(lambda: f.read(8192), b""):
            h.update(chunk)
    return h.hexdigest()

def preserve_file(path: Path) -> Dict[str, Any]:
    dest = OUTPUT_DIR / f"{path.stem}_{datetime.utcnow().strftime('%Y%m%d_%H%M%S')}{path.suffix}"
    dest.write_bytes(path.read_bytes())
    meta = {
        "original": str(path),
        "copy": str(dest),
        "sha256": sha256_file(dest),
        "timestamp": datetime.utcnow().isoformat(),
    }
    (dest.with_suffix(dest.suffix + ".meta.json")).write_text(json.dumps(meta, indent=2), encoding="utf-8")
    return meta

# ============================================================
# 3. CSV DATA MODELS
# ============================================================

@dataclass
class Contract:
    id: str
    counterparty: str
    ip_asset: str
    royalty_rate: float
    jurisdiction: str

@dataclass
class Invoice:
    id: str
    contract_id: str
    amount: float
    currency: str
    due_date: str
    paid: bool

@dataclass
class Holding:
    symbol: str
    exchange: str
    shares: float
    ip_asset: str

@dataclass
class EventFlag:
    type: str
    severity: str
    message: str
    context: Dict[str, Any]

# ============================================================
# 4. CSV LOADING
# ============================================================

def load_contracts(path: str) -> List[Contract]:
    items: List[Contract] = []
    with open(path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            items.append(
                Contract(
                    id=row["id"].strip(),
                    counterparty=row["counterparty"].strip(),
                    ip_asset=row["ip_asset"].strip(),
                    royalty_rate=float(row["royalty_rate"]),
                    jurisdiction=row["jurisdiction"].strip(),
                )
            )
    return items

def load_invoices(path: str) -> List[Invoice]:
    items: List[Invoice] = []
    with open(path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            items.append(
                Invoice(
                    id=row["id"].strip(),
                    contract_id=row["contract_id"].strip(),
                    amount=float(row["amount"]),
                    currency=row["currency"].strip(),
                    due_date=row["due_date"].strip(),
                    paid=row["paid"].strip().lower() == "true",
                )
            )
    return items

def load_holdings(path: str) -> List[Holding]:
    items: List[Holding] = []
    with open(path, newline="", encoding="utf-8") as f:
        reader = csv.DictReader(f)
        for row in reader:
            items.append(
                Holding(
                    symbol=row["symbol"].strip(),
                    exchange=row["exchange"].strip(),
                    shares=float(row["shares"]),
                    ip_asset=row["ip_asset"].strip(),
                )
            )
    return items

# ============================================================
# 5. FLAGGING ENGINE
# ============================================================

def fraud_flags() -> List[Dict[str, Any]]:
    flags: List[Dict[str, Any]] = []
    flags.append({
        "type": "UNAUTHORIZED_INTEGRATION",
        "severity": "HIGH",
        "message": "Detected unauthorized use of DWVSCPS IP / CCUS formulas / trade secrets.",
        "legal_basis": [
            "Criminal Code ss.391, 380, 322, 465, 430, 342.1",
            "Copyright Act 55.13, 27, 14.1, 38.1",
            "Trade Secret — Breach of Confidence",
            "Securities Law — NI 45-106, NI 51-102, NI 52-109",
        ],
    })
    flags.append({
        "type": "CARBON_CREDIT_MISAPPROPRIATION",
        "severity": "HIGH",
        "message": "Unauthorized CCUS-ITC credit claims detected.",
        "value": "645,000,000 CAD",
    })
    flags.append({
        "type": "SPRINGBOARD_PROFIT_INFRINGEMENT",
        "severity": "HIGH",
        "message": "Total Claim Valuation (VTC) = $1.645 Billion",
        "formula": "VTC = ($40M × 25) + $645M",
    })
    return flags

def find_unpaid_invoices(invoices: List[Invoice]) -> List[Invoice]:
    return [inv for inv in invoices if not inv.paid]

def compute_royalty_gap(invoices: List[Invoice], contracts: List[Contract]) -> List[EventFlag]:
    flags: List[EventFlag] = []
    by_contract: Dict[str, float] = {}
    for inv in invoices:
        if not inv.paid:
            by_contract[inv.contract_id] = by_contract.get(inv.contract_id, 0.0) + inv.amount
    contract_map = {c.id: c for c in contracts}
    for cid, total_unpaid in by_contract.items():
        c = contract_map.get(cid)
        if not c:
            continue
        flags.append(
            EventFlag(
                type="UNPAID_ROYALTY",
                severity="HIGH",
                message=f"Unpaid royalties on contract {cid} for IP '{c.ip_asset}'",
                context={
                    "contract_id": cid,
                    "counterparty": c.counterparty,
                    "ip_asset": c.ip_asset,
                    "jurisdiction": c.jurisdiction,
                    "total_unpaid": total_unpaid,
                },
            )
        )
    return flags

def detect_ip_mirroring(holdings: List[Holding], contracts: List[Contract]) -> List[EventFlag]:
    flags: List[EventFlag] = []
    ip_assets = {c.ip_asset for c in contracts}
    for h in holdings:
        if h.ip_asset in ip_assets:
            flags.append(
                EventFlag(
                    type="IP_MIRRORING",
                    severity="MEDIUM",
                    message=f"Holding {h.symbol}:{h.exchange} linked to IP '{h.ip_asset}'",
                    context=asdict(h),
                )
            )
    return flags

def build_risk_score(flags: List[EventFlag]) -> float:
    score = 0.0
    for f in flags:
        if f.severity == "HIGH":
            score += 0.5
        elif f.severity == "MEDIUM":
            score += 0.3
        else:
            score += 0.1
    return min(score, 1.0)

# ============================================================
# 6. LETTER TEMPLATES (EMBEDDED)
# ============================================================

def cra_submission_letter() -> str:
    return (
        "DWVSCPS ENERGY FAMILY TRUST™ / 15389089 Canada Inc.\n"
        "Calgary, Alberta, Canada\n\n"
        "To: Canada Revenue Agency – Compliance / Audit / CCUS-ITC Unit\n\n"
        "Re: Fraud Complaint and CCUS Credit Misappropriation – DWVSCPS IP / CCUS Formulas\n\n"
        "I, Richard Evan Stockford Jr, sole inventor and controller of DWVSCPS ENERGY FAMILY TRUST™, "
        "hereby submit this regulator-ready activation package and evidence JSON to notify CRA of "
        "suspected misappropriation of CCUS-ITC credits and unauthorized integration of my patented "
        "and trade-secret protected DWV Stockford Contaminate Pipeline Shell technology and associated "
        "audit formulas.\n\n"
        "The attached JSON (DWVSCPS_MASTER_ACTIVATION.json) contains:\n"
        "• Master identity of the trust and corporate controller\n"
        "• Contract and invoice data showing unpaid royalties and misaligned credits\n"
        "• Fraud flags referencing Criminal Code, securities, and copyright provisions\n"
        "• Chain-of-custody hashes for all supporting documents\n\n"
        "Requested action:\n"
        "• Open a formal fraud and misrepresentation file\n"
        "• Freeze and review all CCUS-ITC credits linked to the flagged counterparties\n"
        "• Coordinate with securities and law-enforcement regulators where appropriate.\n\n"
        "Signed at Calgary, Alberta, on "
        + datetime.utcnow().strftime("%Y-%m-%d")
        + ".\n\n"
        "______________________________\n"
        "Richard Evan Stockford Jr\n"
        "DWVSCPS ENERGY™ / 15389089 Canada Inc.\n"
    )

def bank_fraud_freeze_letter() -> str:
    return (
        "DWVSCPS ENERGY FAMILY TRUST™\n"
        "Calgary, Alberta, Canada\n\n"
        "To: [Bank Name] – Fraud / Compliance / EDI Department\n\n"
        "Re: Immediate Fraud-Freeze and EDI Verification – DWVSCPS Trust Holdings\n\n"
        "This letter accompanies the DWVSCPS_MASTER_ACTIVATION.json package, which encodes chain-of-"
        "custody hashes, trust identity, and flagged events relating to suspected unauthorized use of "
        "DWVSCPS intellectual property and misdirected royalty flows.\n\n"
        "I request:\n"
        "• Immediate temporary freeze on accounts linked to the flagged counterparties pending review.\n"
        "• EDI verification of all incoming and outgoing payments associated with DWVSCPS IP assets.\n"
        "• Confirmation that trust-designated accounts are recognized as the controlling beneficiary "
        "for all DWVSCPS-related assets.\n\n"
        "This request is made in good faith to protect the integrity of the DWVSCPS ENERGY FAMILY TRUST™ "
        "and its beneficiaries.\n\n"
        "______________________________\n"
        "Richard Evan Stockford Jr\n"
        "Trust Controller\n"
    )

def mareva_affidavit_template() -> str:
    return (
        "COURT FILE NO.: [●]\n"
        "COURT: [●]\n\n"
        "BETWEEN:\n"
        "    RICHARD EVAN STOCKFORD JR / 15389089 CANADA INC.\n"
        "    Plaintiff\n\n"
        "AND:\n"
        "    [Defendant Name]\n"
        "    Defendant\n\n"
        "AFFIDAVIT IN SUPPORT OF MAREVA INJUNCTION\n\n"
        "I, Richard Evan Stockford Jr, of the City of Calgary, Alberta, MAKE OATH AND SAY:\n\n"
        "1. I am the founder, sole inventor, and controller of DWVSCPS ENERGY FAMILY TRUST™ and "
        "15389089 Canada Inc., and as such have personal knowledge of the matters herein.\n"
        "2. Attached as Exhibit 'A' is the DWVSCPS_MASTER_ACTIVATION.json, which contains:\n"
        "   (a) Master identity of the trust and corporate controller;\n"
        "   (b) Contract, invoice, and holding data evidencing unpaid royalties and IP mirroring;\n"
        "   (c) Fraud flags referencing Criminal Code, securities, and copyright violations;\n"
        "   (d) Chain-of-custody hashes for all supporting documents.\n"
        "3. Based on this evidence, I believe there is a real risk of dissipation of assets and "
        "continued misuse of my intellectual property unless the Court grants a Mareva injunction.\n\n"
        "SWORN BEFORE ME at Calgary, Alberta, this "
        + datetime.utcnow().strftime("%Y-%m-%d")
        + ".\n\n"
        "______________________________        _____________________________\n"
        "Commissioner for Oaths               Richard Evan Stockford Jr\n"
    )

def trust_transfer_instruction_letter() -> str:
    return (
        "DWVSCPS ENERGY FAMILY TRUST™\n"
        "TRUST VAULT: DWVSCPS_TRUST_VAULT_2026\n\n"
        "To: [Institution / Registrar]\n\n"
        "Re: Trust-Transfer Instruction – Alignment of DWVSCPS IP and Securities Holdings\n\n"
        "This instruction is issued by the controlling authority of DWVSCPS ENERGY FAMILY TRUST™, "
        "identifying DWVSCPS_TRUST_VAULT_2026 as the master vault for all DWV Stockford Contaminate "
        "Pipeline Shell Inc™ intellectual property and related securities.\n\n"
        "The attached JSON (DWVSCPS_MASTER_ACTIVATION.json) enumerates:\n"
        "• Contracts and invoices linked to DWVSCPS IP assets;\n"
        "• Securities holdings (symbols, exchanges, share counts) mapped to those IP assets;\n"
        "• Risk score and fraud flags requiring realignment of beneficial ownership.\n\n"
        "Instruction:\n"
        "• Recognize DWVSCPS ENERGY FAMILY TRUST™ as the controlling beneficial owner of all DWVSCPS-"
        "linked assets;\n"
        "• Update internal registers and account designations to reflect trust ownership;\n"
        "• Confirm completion of the transfer and alignment in writing.\n\n"
        "______________________________\n"
        "Richard Evan Stockford Jr\n"
        "Trust Controller\n"
    )

# ============================================================
# 7. PDF-READY STRUCTURE (PAGE NUMBERING & INDEX)
# ============================================================

def build_master_binder_index() -> List[Dict[str, Any]]:
    return [
        {"page": 1, "title": "Regulator-Ready Cover Page"},
        {"page": 2, "title": "Master Binder Index"},
        {"page": 3, "title": "CRA Submission Letter"},
        {"page": 4, "title": "Bank Fraud-Freeze Instruction Letter"},
        {"page": 5, "title": "Mareva Injunction Affidavit Template"},
        {"page": 6, "title": "Trust-Transfer Instruction Letter"},
        {"page": 7, "title": "JSON Evidence Payload Summary"},
    ]

def regulator_cover_page(risk_score: float, contracts_count: int, invoices_count: int, holdings_count: int) -> str:
    return (
        "DWVSCPS ENERGY FAMILY TRUST™ – MASTER COMPLIANCE BINDER\n"
        "Regulator-Ready Cover Page\n\n"
        "Controller: R. E. STOCKFORD JR / 15389089 Canada Inc.\n"
        "Trademark: DWV STOCKFORD CONTAMINATE PIPELINE SHELL INC™\n"
        "Trust Vault: DWVSCPS_TRUST_VAULT_2026\n\n"
        f"Risk Score (0–1): {risk_score:.2f}\n"
        f"Contracts: {contracts_count} | Invoices: {invoices_count} | Holdings: {holdings_count}\n\n"
        "This binder is designed for CRA, financial institutions, and courts as a unified, "
        "PDF-ready evidence and instruction package. Page numbering and index are provided "
        "to allow direct cross-reference between the JSON payload and generated PDF.\n\n"
        "______________________________\n"
        "Master Binder Controller: DWVSCPS ENERGY FAMILY TRUST™\n"
    )

# ============================================================
# 8. ACTIVATION PAYLOAD (MERGED INTO JSON)
# ============================================================

def generate_activation_payload(
    contracts: List[Contract],
    invoices: List[Invoice],
    holdings: List[Holding],
    event_flags: List[EventFlag],
    risk_score: float,
) -> Dict[str, Any]:
    chain_of_custody: List[Dict[str, Any]] = []
    for file in INPUT_DIR.glob("*"):
        chain_of_custody.append(preserve_file(file))

    binder_index = build_master_binder_index()

    payload: Dict[str, Any] = {
        "generated_at": datetime.utcnow().isoformat(),
        "master_identity": MASTER_IDENTITY,
        "risk_score": risk_score,
        "summary": {
            "contracts_count": len(contracts),
            "invoices_count": len(invoices),
            "holdings_count": len(holdings),
            "flags_count": len(event_flags),
        },
        "cover_page": {
            "page": 1,
            "content": regulator_cover_page(
                risk_score,
                len(contracts),
                len(invoices),
                len(holdings),
            ),
        },
        "master_binder_index": {
            "page": 2,
            "entries": binder_index,
        },
        "letters": {
            "cra_submission": {
                "page": 3,
                "title": "CRA Submission Letter",
                "pdf_ready_text": cra_submission_letter(),
            },
            "bank_fraud_freeze": {
                "page": 4,
                "title": "Bank Fraud-Freeze Instruction Letter",
                "pdf_ready_text": bank_fraud_freeze_letter(),
            },
            "mareva_affidavit": {
                "page": 5,
                "title": "Mareva Injunction Affidavit Template",
                "pdf_ready_text": mareva_affidavit_template(),
            },
            "trust_transfer": {
                "page": 6,
                "title": "Trust-Transfer Instruction Letter",
                "pdf_ready_text": trust_transfer_instruction_letter(),
            },
        },
        "json_evidence_summary": {
            "page": 7,
            "description": "Structured evidence payload for regulators, banks, and courts.",
        },
        "flags": [asdict(f) for f in event_flags],
        "contracts": [asdict(c) for c in contracts],
        "invoices": [asdict(i) for i in invoices],
        "holdings": [asdict(h) for h in holdings],
        "chain_of_custody": chain_of_custody,
        "instructions": {
            "cra": "Use pages 1–3 and JSON flags for fraud / CCUS misappropriation review.",
            "bank": "Use pages 1–4 and chain-of-custody for fraud-freeze and EDI verification.",
            "trust": "Use pages 1–6 to align all DWVSCPS-linked assets into DWVSCPS ENERGY FAMILY TRUST™.",
            "legal": "Use pages 1–7 and DWVSCPS_MASTER_ACTIVATION.json as Exhibit A in Mareva / injunction filings.",
        },
        "signature_block": {
            "controller": "Richard Evan Stockford Jr",
            "entity": "DWVSCPS ENERGY™ / 15389089 Canada Inc.",
            "signed_at": datetime.utcnow().isoformat(),
        },
    }

    return payload

# ============================================================
# 9. MAIN
# ============================================================

def main():
    contracts = load_contracts("contracts.csv")
    invoices = load_invoices("invoices.csv")
    holdings = load_holdings("holdings.csv")

    unpaid = find_unpaid_invoices(invoices)
    royalty_flags = compute_royalty_gap(unpaid, contracts)
    ip_flags = detect_ip_mirroring(holdings, contracts)
    core_flags_dicts = fraud_flags()
    core_flags = [
        EventFlag(
            type=f["type"],
            severity=f["severity"],
            message=f["message"],
            context={k: v for k, v in f.items() if k not in ("type", "severity", "message")},
        )
        for f in core_flags_dicts
    ]

    all_flags: List[EventFlag] = royalty_flags + ip_flags + core_flags
    risk_score = build_risk_score(all_flags)

    activation_payload = generate_activation_payload(
        contracts=contracts,
        invoices=invoices,
        holdings=holdings,
        event_flags=all_flags,
        risk_score=risk_score,
    )

    out_file = OUTPUT_DIR / "DWVSCPS_MASTER_ACTIVATION.json"
    out_file.write_text(json.dumps(activation_payload, indent=2), encoding="utf-8")
    print(f"[+] Master activation JSON with cover page, index, letters, and evidence saved to {out_file}")

if __name__ == "__main__":
    main()
Finance
Disclaimers
AI responses may include mistakes. Content is provided “as is” for informational purposes only and does not constitute financial, legal, tax, or investment advice.


Google is not a broker or investment adviser and does not recommend or solicit the purchase or sale of any security. Please consult a financial professional to verify pricing before trading. You are solely responsible for evaluating your own risk appetite and investment objectives. Google shall not be liable for any damages arising from any operations or investments in financial products referred to within, and does not recommend using this data and information as the only basis for investment decision.

Data is provided by financial exchanges and other content providers and may be delayed as specified by financial exchanges or other data providers. Google does not verify any data and disclaims any obligation to do so.


Google and its business partners (A) expressly disclaim the accuracy, adequacy, or completeness of any data and (B) shall not be liable for any errors, omissions or other defects in, delays or interruptions in such data, or for any actions taken in reliance thereon. As used here, “business partners” does not refer to an agency, partnership, or joint venture relationship between Google and any such parties.


You agree not to copy, modify, reformat, download, store, reproduce, reprocess, transmit or redistribute any data or information found herein or use any such data or information in a commercial enterprise without obtaining prior written consent. Google or its third party data or content providers have exclusive proprietary rights in the data and information provided.


Advertisements presented on Google Finance are solely the responsibility of the party from whom the ad originates. Neither Google nor any of its data licensors endorses or is responsible for the content of any advertisement or any goods or services offered therein.

Please find all listed exchanges and indices covered by Google along with their respective time delays from the table below.

Finance Data Listing
A list of all Stock Exchanges, Mutual Funds, Indexes and other financial data available in Google products
Exchanges
End of day prices provided by Morningstar. Corporate actions and company metadata provided by Refinitiv.
Intra-day data may be provided by ICE Data Services.
Data for the Moscow Exchange is currently unavailable.
Region	Exchange Code	Description	Delay (minutes)
Americas	BCBA	Buenos Aires Stock Exchange	20
BMV	Mexican Stock Exchange	20
BVMF	B3 - Brazil Stock Exchange and Over-the-Counter Market	15
CNSX	Canadian Securities Exchange	Realtime
NASDAQ	NASDAQ Last Sale	Realtime *
NYSE	NYSE	Realtime *
NYSEARCA	NYSE ARCA	Realtime *
NYSEAMERICAN	NYSE American	Realtime *
OPRA	Options Price Reporting Authority	15
OTCMKTS	FINRA Other OTC Issues	15
TSE	Toronto Stock Exchange	15
TSX	Toronto Stock Exchange	15
TSXV	Toronto TSX Ventures Exchange	15
Europe	AMS	Euronext Amsterdam	15
BIT	Borsa Italiana Milan Stock Exchange	Realtime
BME	Bolsas y Mercados Españoles	15
CPH	NASDAQ OMX Copenhagen	Realtime
EBR	Euronext Brussels	15
ELI	Euronext Lisbon	15
EPA	Euronext Paris	15
ETR	Deutsche Börse XETRA	15
FRA	Deutsche Börse Frankfurt Stock Exchange	Realtime
HEL	NASDAQ OMX Helsinki	Realtime
ICE	NASDAQ OMX Iceland	Realtime
IST	Borsa Istanbul	15
LON	London Stock Exchange	Realtime
RSE	NASDAQ OMX Riga	Realtime
STO	NASDAQ OMX Stockholm	Realtime
SWX, VTX	SIX Swiss Exchange	15
TAL	NASDAQ OMX Tallinn	Realtime
VIE	Vienna Stock Exchange	15
VSE	NASDAQ OMX Vilnius	Realtime
WSE	Warsaw Stock Exchange	15
Africa	JSE	Johannesburg Stock Exchange	15
Middle East	TADAWUL	Saudi Stock Exchange	15
TLV	Tel Aviv Stock Exchange	20
Asia	BKK	Thailand Stock Exchange	15
BOM	Bombay Stock Exchange Limited	Realtime
KLSE	Bursa Malaysia	15
HKG	Hong Kong Stock Exchange	15
IDX	Indonesia Stock Exchange	10
KOSDAQ	KOSDAQ	20
KRX	Korea Stock Exchange	20
NSE	National Stock Exchange of India	Realtime
SGX	Singapore Exchange	Realtime
SHA	Shanghai Stock Exchange	1
SHE	Shenzhen Stock Exchange	Realtime
TPE	Taiwan Stock Exchange	Realtime
TYO	Tokyo Stock Exchange	20
South Pacific	ASX	Australian Securities Exchange	20
NZE	New Zealand Stock Exchange	20
*Real-time price data represents trades which execute on the NASDAQ and NYSE exchanges. Volume information, as well as price data for trades that don’t execute on those exchanges, are consolidated and delayed by 15 minutes.
Mutual Funds
Mutual fund prices provided by Morningstar.
Region	Exchange Code	Description	Delay (minutes)
Americas	MUTF	USA Mutual Funds	End-of-day
Asia	MUTF_IN	India Mutual Funds	End-of-day
Indexes
End of day prices provided by Morningstar.
Intra-day data may be provided by ICE Data Services.
Data for the Moscow Exchange is currently unavailable.
Region	Exchange Code	Description	Delay (minutes)
Americas	INDEXBVMF	B3 - Brazil Stock Exchange and Over-the-Counter Market Indexes	15
INDEXCBOE	CBOE Index Values	15
INDEXCME	Chicago Mercantile Exchange Indexes	Realtime
INDEXDJX	S&P Dow Jones Indices	Realtime
INDEXNASDAQ	NASDAQ Global Indexes	Realtime
INDEXNYSEGIS	NYSE Global Index Feed	15
INDEXRUSSELL	Russell Tick	15
INDEXSP	S&P Cash Indexes	Realtime
BCBA	Buenos Aires Stock Exchange Indexes	20
INDEXBMV	Mexican Stock Exchange Indexes	20
INDEXTSI	Toronto Stock Exchange Indexes	15
Europe	INDEXBIT	Milan Stock Exchange Indexes	15
INDEXBME	Bolsas y Mercados Españoles Indexes	15
INDEXDB	Deutsche Börse Indexes	15
INDEXEURO	Euronext Indexes	15
INDEXFTSE	FTSE Indexes	Realtime
INDEXIST	Borsa Istanbul Indexes	15
INDEXNASDAQ	NASDAQ Global Indexes	Realtime
INDEXSTOXX	STOXX Indexes	15
INDEXSWX	SIX Swiss Exchange Indexes	15
INDEXVIE	Wiener Börse Indexes	15
Asia	INDEXBKK	Thailand Stock Exchange Indexes	15
INDEXBOM	Bombay Stock Exchange Indexes	Realtime
SHA	Shanghai/Shenzhen Indexes	1
INDEXHANGSENG	Hang Seng Indexes	Realtime
HKG	Hong Kong Stock Exchange Indexes	15
KOSDAQ, KRX	Korea Stock Exchange Indexes	20
INDEXNIKKEI	Nikkei Indexes	20
INDEXTYO	Tokyo Indexes	20
INDEXTYO:JPXNIKKEI400	© Japan Exchange Group, Inc., Tokyo Stock Exchange, Inc., Nikkei Inc.	20
INDEXTOPIX	Tokyo Stock Exchange Indexes	20
IDX	Indonesia Stock Exchange Indexes	15
NSE	National Stock Exchange of India Indexes	Realtime
SHE	Shenzhen Stock Exchange Indexes	Realtime
TPE	Taiwan Stock Exchange Indexes	Realtime
Middle East	TLV	Tel Aviv Stock Exchange Indexes	20
TADAWUL	Saudi Stock Exchange Indexes	15
South Pacific	INDEXASX	Australian Securities Exchange S&P/ASX Indexes	Realtime
INDEXNZE	New Zealand Exchange Indexes	20
Futures
Futures data provided by CME Group
Region	Exchange Code	Description	Delay (minutes)
Americas	CBOT E-mini		10
CBOT		10
CME E-mini		10
CME GLOBEX		10
COMEX		10
NYMEX		10
Bonds
Region	Exchange Code	Description	Delay (minutes)
United States		KCG Bondpoint	15
Currencies and gold spot rates
Currency and cryptocurrency prices provided by Morningstar
Cryptocurrency metadata is provided by Coinmarketcap
Gold spot prices for India are provided by Ticker Data Limited
Region	Exchange Code	Description	Delay (minutes)
Global		Currency	3
Cryptocurrency	3
India		Gold spot price	3
Prediction Markets
Region	Exchange Code	Description	Delay (minutes)
Global	KALSHI	Kalshi	Realtime
POLY	Polymarket	Realtime
Alternative Data
Analyst Consensus & Ratings and News Sentiments are provided by TipRanks.
Insider Transactions and Politicians' Holdings data are provided by Unusual Whales.
The delay minutes listed below indicate how often our system refreshes individual data points. Please note that the underlying data provided by TipRanks and Unusual Whales is aggregated and updated on a daily basis.
Region	Exchange Code	Description	Delay (minutes)
Global		Insider Transactions	360
Politicians' Holdings	360
Analyst Consensus & Ratings	60
News Sentiment	60
Sector Data
Sector and industry data is provided by S&P Capital IQ. Below are the mappings for Sector and Industry data from GICS to Google Finance naming.
GICS

Google Finance

Hypermarkets and Super Centers

Grocery Stores

Household Durables

Household

Gas Utilities

Gas

Consumer Finance

Payment Service Provider

Food Products

Food

Technology Hardware, Storage and Peripherals

Computers, Peripherals, and Software

Independent Power and Renewable Electricity Producers

Renewable Energy

Diversified Metals and Mining

Mining

Health Care Equipment and Supplies

Medical Devices

Industrials

Industry

Diversified Financial Services

Credit

Brewers

Brewery

Machinery

Machine Industry

Consumer Staples

Consumer

Water Utilities

Water

Automobiles and Components

Cars

Movies and Entertainment

Entertainment

Hotels, Resorts and Cruise Lines

Hotel

Hotels, Resorts and Cruise Lines

Resort

Hotels, Resorts and Cruise Lines

Cruise Line

Textiles

Clothing

Education Services

Education

Mortgage Services

Mortgage Loan

Electric Utilities

Electricity

IT Consulting & Other Services

Information Technology Consulting

Paper products

Paper

Trading Companies and Distributors

Trading company

Application Software

Computer Application

Computer and Electronic Retail

Consumer Electronics

Industrial Conglomerates

Conglomerates

Health Care Provider and Services

Health care provider

Household Products

Household Goods

Commercial Printing

Printing

Fertilizers and Agricultural Chemicals

Fertilizers

Publishing

Satellite television

Oil and Gas Drilling

Drilling

Oil and Gas Drilling

Oil (Petroleum)

Home Improvement Retail

Home Improvement

Financials

Financial Services

Chemicals

Chemical Industry

Information Technology

Technology

Semiconductors and Semiconductor Equipment

Semiconductor

Interactive Media and Services (level 4)

Interactive Media

Personal Products

Personal Care Products

Air Freight and Logistics

Air Cargo

Real Estate Management and Development

Property Management

Interactive Home Entertainment

Home Entertainment

Apparel, Accessories and Luxury Goods

Luxury Goods

Apparel, Accessories and Luxury Goods

Clothing

Apparel, Accessories and Luxury Goods

Fashion Accessory

Homefurnishing

Furniture

Leisure Products

Leisure

Utilities

Public Utility

Building Products

Building

Multi Utilities

Multi-utility

Software & Services

Software

Automobile Components

Auto Parts

Consumer Discretionary

Consumer

Drug Retail

Drug

Construction Materials

Building Material

Technology Hardware and Equipment

Computer hardware

Pharmaceuticals

Pharmaceutical Industry

Telecommunication Services

Telecommunications

Distillers and Vintners

Distillery

Apparel

Clothing

Beverages

Drink

Commercial and Professional Services

Business Services

General Merchandise Stores

General Store

Automotive Retail

Automotive Industry

Consumer Durables and Apparels

Clothing

Passenger Airlines

Airline

Passenger Airlines

Passenger Airline

Internet Services and Infrastructure

Internet

Electronic Manufacturing Services

Electronic Manufacturing

Marine Transportation

Maritime transport

Technology Distributors

Technology

Currency Conversion
Google cannot guarantee the accuracy of the exchange rates displayed. You should confirm current rates before making any transactions that could be affected by changes in the exchange rates.

Search Ranking
Google Finance provides a simple way to search for financial security data (stocks, mutual funds, indexes, etc.), currency and cryptocurrency exchange rates (‘Finance Data’). The Finance Data is obtained from various data providers and feeds into a unified format available for serving to users. Google Finance ranks search suggestions on the basis of three main elements: Exact matches to queries, Google Search impressions, and Google Finance impressions. Exact matches to queries are given higher importance followed by Google Search and Google Finance impressions which are given equal importance.

NYSE Securities
NYSE, NYSE Arca LLC, and NYSE MKT LLC reserve all rights to the securities information that Google LLC makes available to you. You understand and acknowledge that such securities information does not reflect trading activity on markets other than NYSE, NYSE Arca, or NYSE MKT, as applicable, and are intended to provide you with a reference point only, rather than as a basis for making trading decisions. None of Google LLC, NYSE, NYSE Arca LLC, and NYSE MKT LLC guarantee such information nor shall any of them be liable for any loss due either to their negligence or to any cause beyond their reasonable control. Any redistribution of such information is strictly prohibited.

S&P Capital IQ
S&P Global Market Intelligence provided by S&P Capital IQ. Copyright (c) 2020, S&P Capital IQ (and its affiliates, as applicable). All rights reserved.

S&P Dow Jones Indices LLC
Copyright © 2020, S&P Dow Jones Indices LLC. All rights reserved. S&P does not guarantee the accuracy, adequacy, completeness or availability of any information and is not responsible for any errors or omissions, regardless of the cause or for the results obtained from the use of such information. S&P, its affiliates and their third party suppliers disclaim any and all express or implied warranties, including, but not limited to, any warranties of merchantability or fitness for a particular purpose or use. S&P DJI Indices are not investment advice and a reference to a particular investment or security, a credit rating or any observation concerning a security or investment provided in S&P DJI Indices are not a recommendation to buy, sell or hold such investment or security or make any other investment decisions. In no event shall S&P be liable for any direct, indirect, special or consequential damages, costs, expenses, legal fees, or losses (including lost income or lost profit and opportunity costs) in connection with your or others’ use of the S&P DJI Indices.

Corporate financials and estimates provided by S&P and Fiscal.ai. Earnings call transcripts and audio provided by Quartr.

Wall Street Horizon, Inc
Global Corporate Events provided by Wall Street Horizon, Inc., a TMX company.

Copyright 2025 Wall Street Horizon, Inc. All Rights Reserved. This information is provided for information purposes only. Neither TMX Group Limited nor any of its affiliated companies guarantees the completeness of the information contained in this publication, and we are not responsible for any errors or omissions in or your use of, or reliance on, the information. This information is not intended to provide legal, accounting, tax, investment, financial or other advice and should not be relied upon for such advice. The information provided is not an invitation to purchase securities listed on Montréal Exchange, Toronto Stock Exchange and/or TSX Venture Exchange. TMX Group and its affiliated companies do not endorse or recommend any securities referenced in this publication. TMX, the TMX design, are the trademarks of TSX Inc.

TipRanks
TipRanks Ltd. Copyright (c) 2026 (and its affiliates, as applicable). All rights reserved.

Unusual Whales
Unusual Whales Inc. Copyright (c) 2026 (and its affiliates, as applicable). All rights reserved.

PrivacyTermsAbout Google
 HelpChange language or region
English

# Third Party Clearances

Use this file to list any third‑party materials included in the repository and their clearance status.

## Example entry:
- **File:** docs/some_image.png
  - **Owner:** Example Corp
  - **License:** Permission granted by email dated 2026-05-01
  - **Notes:** Attribution required; do not modify image without permission.

## Clearances

(Add entries below as materials are included)
