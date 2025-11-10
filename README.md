# Flight Booking Auto-Fill Userscript

**Version:** 1.5  
**Author:** Amos Kiprotich  
**Website:** [https://dlang.benfex.net](https://dlang.benfex.net)  

---

## Description

This userscript automates filling flight booking forms on Flynas CRM for one booking at a time. It saves agents time by auto-populating passenger info, booking details, and payment fields from a JSON data source.  

It also allows manual confirmation and selecting the reason for travel. Group bookings and multiple passengers are supported.  

---

## Features

- Auto-fills **passenger information**:
  - Full Name
  - ID/Passport Number (`00` by default)
  - Phone Number (`00` by default)
  - Reason for Travel (prompted once per booking, applied to all passengers)
- Auto-fills **booking details**:
  - Sales Type (`Individual` or `Group`)
  - Sales Origin (`Online`)
  - Seat Type (`Economy`)
  - Flight Date (constant: `3rd October 2025`)
  - Flight Time (constant: `04:00 AM`)
  - Agency Name and Account (if provided)
- Auto-fills **payment information**
- Supports **group bookings**
- Hotkey: **Ctrl + N** to load the next booking
- Manual confirmation before submitting each booking

---

## Installation

1. Install a userscript manager I reccomend **Violentmonkey** 
2. Copy the full script below and create a new script in the manager.
3. Save it and navigate to:  https://crm.flynas.co.ke/bookings/create/new


---

## Full Script

```javascript
// ==UserScript==
// @name         Flight Booking Auto-Fill
// @namespace    https://dlang.benfex.net/
// @version      1.5
// @description  Auto-fills one booking at a time
// @match        https://crm.flynas.co.ke/bookings/create/new
// @grant        GM_xmlhttpRequest
// @connect      ngrok-free.app
// @require      https://code.jquery.com/jquery-3.6.0.min.js
// @run-at       document-idle
// ==/UserScript==

(async function () {
 'use strict';
 const DATA_URL = 'https://files.benfex.net/uploads/flynas/3rd.json';
 const $ = window.jQuery;

 // ---------- Constants ----------
 const FLIGHT_TIME = '04:00';
 const FLIGHT_DATE = '2025-10-03';
 const SALES_ORIGIN = 'Online';
 const DEFAULT_SEAT_TYPE = 'economy';

 // ---------- Helper: Fetch JSON ----------
 function fetchJSON(url) {
     return new Promise((resolve, reject) => {
         GM_xmlhttpRequest({
             method: 'GET',
             url,
             onload: res => {
                 try {
                     resolve(JSON.parse(res.responseText));
                 } catch (e) {
                     reject(e);
                 }
             },
             onerror: reject
         });
     });
 }

 // ---------- Helper: Ask reason ----------
 async function askReason(pnr) {
     return new Promise(resolve => {
         const reason = prompt(`Enter reason for PNR ${pnr}:\nOptions: Business, Labour, Other`);
         if (reason?.toLowerCase() === 'other') {
             const manualReason = prompt('Enter manual reason:');
             resolve(manualReason || 'Other');
         } else {
             resolve(reason || 'Other');
         }
     });
 }

 // ---------- Fill Booking ----------
 async function fillBooking(booking) {
     $('select[name="channel_type"]').val(booking.channel_type.toLowerCase()).trigger('change');

     if (booking.agency_name) {
         $('#agencySelect option').filter(function () {
             return $(this).text().toLowerCase().includes(booking.agency_name.toLowerCase());
         }).prop('selected', true).trigger('change');
         $('#agencyNameDisplay').val(booking.agency_name);
         $('#agencyAccountDisplay').val(booking.agency_account || '');
     }

     $('input[name="pnr"]').val(booking.pnr);

     $('#flightSelect option').filter(function () {
         return $(this).data('number') === booking.flight_number;
     }).prop('selected', true).trigger('change');

     $('#departureTime').val(FLIGHT_TIME);
     $('#flightDate').val(FLIGHT_DATE);

     const [origin, destination] = booking.route.split('-');
     $('#origin').val(origin);
     $('#destination').val(destination);
     $('#baseFare').val(booking.air_fare);

     const passengerCount = booking.passengers.length;
     $('#passengerCount').val(passengerCount);
     $('#salesType').val(passengerCount > 1 ? 'group' : 'individual').trigger('change');
     $('select[name="seat_type"]').val(DEFAULT_SEAT_TYPE);
     $('input[name="sales_origin"]').val(SALES_ORIGIN);
     $('input[name="call_centre_agent"]').val(booking.call_centre_agent || 'Amos Langat');

     const reasonForTravel = await askReason(booking.pnr);

     const container = $('#passengersContainer');
     container.empty();
     booking.passengers.forEach((p, idx) => {
         const passengerHtml = `
         <div class="passenger-form mb-6 pb-6 border-b border-purple-200">
             <h4 class="font-bold text-purple-700 mb-4">Passenger ${idx + 1}</h4>
             <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
                 <div>
                     <label class="block text-sm font-bold text-gray-700 mb-2">Full Name *</label>
                     <input type="text" name="passengers[${idx}][name]" value="${p.name}" required class="w-full rounded-xl border-gray-300 focus:border-purple-500 focus:ring-purple-500">
                 </div>
                 <div>
                     <label class="block text-sm font-bold text-gray-700 mb-2">ID/Passport Number *</label>
                     <input type="text" name="passengers[${idx}][id_number]" value="00" required class="w-full rounded-xl border-gray-300 focus:border-purple-500 focus:ring-purple-500">
                 </div>
                 <div>
                     <label class="block text-sm font-bold text-gray-700 mb-2">Phone Number *</label>
                     <input type="text" name="passengers[${idx}][phone]" value="00" required class="w-full rounded-xl border-gray-300 focus:border-purple-500 focus:ring-purple-500">
                 </div>
                 <div>
                     <label class="block text-sm font-bold text-gray-700 mb-2">Reason for Travel *</label>
                     <input type="text" name="passengers[${idx}][reason_for_travel]" value="${reasonForTravel}" required class="w-full rounded-xl border-gray-300 focus:border-purple-500 focus:ring-purple-500">
                 </div>
             </div>
         </div>`;
         container.append(passengerHtml);
     });

     $('#ticketAmount').val(booking.air_fare);
     $('#commissionAmount').val(booking.commission);
     const totalAmount = booking.total_amount + (booking.admin_fee || 0);
     $('#totalAmountHidden').val(totalAmount);
     $('#totalAmountDisplay').val(totalAmount);
     $('#adminFee').val(booking.admin_fee || 0);
     $('#adminFeeDisplay').val(`$${(booking.admin_fee || 0).toFixed(2)}`);
     $('#totalPayable').val(totalAmount);

     // Set default payment method
     $('select[name="payment_method"]').val('bank_transfer').trigger('change');
 }

 // ---------- Main ----------
 try {
     const bookings = await fetchJSON(DATA_URL);
     let currentIndex = 0;

     if (bookings.length === 0) {
         alert("No bookings found in JSON.");
         return;
     }

     await fillBooking(bookings[currentIndex]);
     alert(`Booking ${currentIndex + 1} of ${bookings.length} loaded. Confirm and click 'Create Booking'.`);

     $(document).on('keydown', async function (e) {
         if (e.ctrlKey && e.key === 'n') {
             e.preventDefault();
             currentIndex++;
             if (currentIndex < bookings.length) {
                 await fillBooking(bookings[currentIndex]);
                 alert(`Booking ${currentIndex + 1} of ${bookings.length} loaded. Confirm and click 'Create Booking'.`);
             } else {
                 alert('All bookings processed.');
             }
         }
     });

 } catch (err) {
     console.error("Error fetching or parsing JSON:", err);
     alert("Failed to load bookings JSON. Check console for details.");
 }
})();

