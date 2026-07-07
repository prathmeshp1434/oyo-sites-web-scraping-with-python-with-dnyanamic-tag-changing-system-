# oyo-sites-web-scraping-with-python-with-dnyanamic-tag-changing-system-

import CSV
import json
import cloudscraper 
from bs4 import BeautifulSoup
import re

url = "https://www.oyorooms.com/hotels-in-aurangabad/"

scraper = cloudscraper.create_scraper()
print("🔓 connecting with server...")
response = scraper.get(url)

if response.status_code == 200:
    print("🤖 Robot entered!")
    soup = BeautifulSoup(response.text, "html.parser")
    hotel_list = []
    
    # === RAASTA 1: JSON KHAZANA ===
    script_tag = soup.find("script", string=re.compile("window.__PRELOADED_STATE__"))
    if script_tag:
        try:
            raw_data = script_tag.string
            json_text = raw_data.split("window.__PRELOADED_STATE__=")[1].strip()
            if json_text.endswith(";"):
                json_text = json_text[:-1]
                
            data = json.loads(json_text)
            search_page = data.get('searchPage', {})
            hotel_list = search_page.get('hotelsData', {}).get("hotels", [])
            
            if not hotel_list:
                for key, value in data.items():
                    if isinstance(value, dict) and 'hotels' in value:
                        hotel_list = value['hotels']
                        break
            if hotel_list:
                print('💎 JSON khajane se maal mil gaya!')
        except Exception:
            pass
            
    # === RAASTA 2: NEW HTML NAKSHA ===
    if not hotel_list:
        print("⚠️ Script tag nahi mila, naye HTML nakshe se dhoondh rahe hain...")
        html_hotels = soup.find_all("div", class_=re.compile(r"hotelCardListing|oyo-row"))
        
        for card in html_hotels:
            name_tag = card.find(["h3", "h1", "span"], class_=re.compile(r"hotelName|heading"))
            address_tag = card.find("span", attrs={"itemprop": "streetAddress"}) or card.find("span", class_=re.compile(r"location|address"))
            price_tag = card.find(class_=re.compile(r"listingPrice|finalPrice|roomPrice"))
            rating_tag = card.find("span", class_=re.compile(r"is-font-bold|rating"))
            
            if name_tag:
                hotel_dict = {
                    "name": name_tag.text.strip(),
                    "address": address_tag.text.strip() if address_tag else "Location nahi mili",
                    "pricing": {"bookingPrice": price_tag.text.strip() if price_tag else "Price nahi mila"},
                    "rating": {"value": rating_tag.text.strip() if rating_tag else "No rating"}
                }
                if hotel_dict not in hotel_list:
                    hotel_list.append(hotel_dict)

    # === MAAL MIL GAYA? AB EXCEL ME LOCK KARO ===
    if hotel_list:
        print(f"\n🎉 Kul mila kar {len(hotel_list)} hotels ka maal mila hai.")
        filename = "aurangabad_oyo_hotels.csv"
        
        with open(filename, mode="w", newline="", encoding="utf-8") as file:
            writer = csv.writer(file)
            writer.writerow(["S.No.", "Hotel Name", "Address", "Price", "Rating"])
            
            for index, hotel in enumerate(hotel_list, 1):
                name = hotel.get("name", "Name nahi mila")
                address = hotel.get("address", "Address nahi mila")
                
                rating_data = hotel.get("rating", {})
                rating = rating_data.get("value", "No rating") if isinstance(rating_data, dict) else "No rating"
                
                pricing_dict = hotel.get("pricing", {})
                price = "Price nahi mila"
                if isinstance(pricing_dict, dict):
                    price = pricing_dict.get("bookingPrice") or hotel.get("roomPrice") or hotel.get("finalPrice") or "Price nahi mila"
                else:
                    price = pricing_dict
                
                # 🌟 PRICE KA RAITA SAAF KARNE WALA BADLAAV YAHAN HAI 🌟
                if '₹' in str(price):
                    parts = str(price).split('₹')
                    print(parts)
                    actual_price = re.findall(r'\d+', parts[1])
                    print(actual_price)
                    clean_price = f"₹{actual_price[0]}" if actual_price else f"₹{parts[1]}"
                    print(clean_price)
                else:
                    actual_price = re.findall(r'\d+', str(price))
                    clean_price = f"₹{actual_price[0]}" if actual_price else "Price nahi mila"
                
                writer.writerow([index, name, address, clean_price, rating])
                
        print(f"📁 Maal ekdam safe hai bhai! Excel file '{filename}' me lock kar diya hai.\n")
    else:
        print("❌ Data nahi mil paya.")
else:
    print("🚫 Entry blocked.")