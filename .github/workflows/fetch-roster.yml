#!/usr/bin/env python3
"""
Pasco Jail Roster Fetcher
Pulls inmate data, charges, and mugshots from Pasco Corrections API
Saves to data/roster.json for WordPress to display
"""

import requests
import json
import os
import base64
import time
from datetime import datetime

BASE_URL = "https://jailinfo.pascocorrections.net/jmcapi"
DATA_DIR = "data"
MUGSHOT_DIR = "data/mugshots"

HEADERS = {
    "User-Agent": "Mozilla/5.0 (compatible; PascoNews/1.0)",
    "Accept": "application/json",
    "Referer": "https://jailinfo.pascocorrections.net/jmc/",
}

def fetch_roster():
    """Fetch all inmates currently in custody"""
    print("Fetching roster...")
    resp = requests.get(f"{BASE_URL}/custody/GetInCustody", headers=HEADERS, timeout=30)
    resp.raise_for_status()
    inmates = resp.json()
    print(f"Found {len(inmates)} inmates")
    return inmates

def fetch_charges(name_id):
    """Fetch charges for a specific inmate"""
    try:
        resp = requests.get(
            f"{BASE_URL}/arrest/GetArrestsByNameId",
            params={"nameId": name_id},
            headers=HEADERS,
            timeout=15
        )
        if resp.status_code == 200:
            return resp.json()
    except Exception as e:
        print(f"  Charges error for {name_id}: {e}")
    return []

def fetch_mugshot(name_id):
    """Fetch mugshot as base64 and save to file"""
    mugshot_path = f"{MUGSHOT_DIR}/{name_id}.jpg"
    
    # Skip if already downloaded
    if os.path.exists(mugshot_path):
        return f"mugshots/{name_id}.jpg"
    
    try:
        resp = requests.get(
            f"{BASE_URL}/custody/GetMugshotByNameId",
            params={"nameId": name_id, "searchAll": "false"},
            headers=HEADERS,
            timeout=15
        )
        if resp.status_code == 200 and resp.content:
            # Response is base64 encoded image data
            content = resp.text.strip().strip('"')
            if content and len(content) > 100:
                img_data = base64.b64decode(content)
                os.makedirs(MUGSHOT_DIR, exist_ok=True)
                with open(mugshot_path, 'wb') as f:
                    f.write(img_data)
                return f"mugshots/{name_id}.jpg"
    except Exception as e:
        print(f"  Mugshot error for {name_id}: {e}")
    return None

def process_charges(raw_charges):
    """Clean and format charge data"""
    charges = []
    for arrest in raw_charges:
        charge_list = arrest.get('charges', [arrest]) if 'charges' in arrest else [arrest]
        for charge in charge_list:
            charges.append({
                'chargeDescription': charge.get('chargeDescription') or charge.get('charge') or charge.get('description') or 'Unknown Charge',
                'statute':           charge.get('statute') or charge.get('statuteCode') or '',
                'chargeType':        charge.get('chargeType') or charge.get('type') or '',
                'bondAmount':        charge.get('bondAmount') or charge.get('bond') or '',
                'caseNumber':        charge.get('caseNumber') or '',
            })
    return charges

def main():
    os.makedirs(DATA_DIR, exist_ok=True)
    os.makedirs(MUGSHOT_DIR, exist_ok=True)

    # Load existing data to avoid re-fetching mugshots
    existing_data = {}
    roster_file = f"{DATA_DIR}/roster.json"
    if os.path.exists(roster_file):
        with open(roster_file, 'r') as f:
            existing = json.load(f)
            for inmate in existing:
                existing_data[inmate['nameId']] = inmate

    # Fetch fresh roster
    inmates = fetch_roster()
    roster = []

    for i, inmate in enumerate(inmates):
        name_id = inmate.get('nameId')
        if not name_id:
            continue

        print(f"[{i+1}/{len(inmates)}] {inmate.get('firstName')} {inmate.get('lastName')} ({name_id})")

        # Get charges
        raw_charges = fetch_charges(name_id)
        charges = process_charges(raw_charges) if raw_charges else []

        # Get mugshot — reuse if we already have it
        if name_id in existing_data and existing_data[name_id].get('mugshot_url'):
            mugshot_url = existing_data[name_id]['mugshot_url']
        else:
            mugshot_url = fetch_mugshot(name_id)
            time.sleep(0.3)  # Be polite to the server

        roster.append({
            'nameId':        name_id,
            'firstName':     inmate.get('firstName', ''),
            'middleName':    inmate.get('middleName', ''),
            'lastName':      inmate.get('lastName', ''),
            'suffix':        inmate.get('suffix', ''),
            'dob':           inmate.get('dob', ''),
            'age':           inmate.get('age', ''),
            'race':          inmate.get('race', ''),
            'sex':           inmate.get('sex', ''),
            'bookingNumber': inmate.get('bookingNumber', ''),
            'bookingDate':   inmate.get('bookingDate', ''),
            'bookingTime':   inmate.get('bookingTime', ''),
            'nmimage':       inmate.get('nmimage', ''),
            'mugshot_url':   mugshot_url,
            'charges':       charges,
        })

    # Save roster
    with open(roster_file, 'w') as f:
        json.dump(roster, f, indent=2)

    print(f"\n✅ Saved {len(roster)} inmates to {roster_file}")
    print(f"📸 Mugshots saved to {MUGSHOT_DIR}/")

    # Save metadata
    with open(f"{DATA_DIR}/meta.json", 'w') as f:
        json.dump({
            'total':     len(roster),
            'updated':   datetime.utcnow().isoformat() + 'Z',
            'source':    'Pasco County Corrections',
        }, f, indent=2)

if __name__ == '__main__':
    main()
