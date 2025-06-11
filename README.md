// === Guna Vision API, full CORS ready ===
// Masukkan API Key Google Vision (kalau guna API key)
// ATAU jika guna Service Account, make sure dah setup di Cloud.

const VISION_API_KEY = "AIzaSyBesfmexK3feou_oP2J-b3lD3PAbQUwqNA"; // <-- EDIT DENGAN API KEY SENDIRI

function doPost(e) {
  // Set CORS supaya frontend GitHub boleh access
  return handleOCR(e);
}

function handleOCR(e) {
  // CORS header
  const corsHeaders = {
    'Access-Control-Allow-Origin': '*',
    'Access-Control-Allow-Methods': 'POST, OPTIONS',
    'Access-Control-Allow-Headers': 'Content-Type'
  };

  try {
    // Terima JSON dari POST (gambar base64)
    let data = JSON.parse(e.postData.contents);
    let imageBase64 = data.image;

    // Panggil Google Vision API (Text Detection)
    const visionUrl = "https://vision.googleapis.com/v1/images:annotate?key=" + VISION_API_KEY;
    const visionPayload = {
      requests: [
        {
          image: { content: imageBase64 },
          features: [{ type: "TEXT_DETECTION" }]
        }
      ]
    };
    const visionRes = UrlFetchApp.fetch(visionUrl, {
      method: "post",
      contentType: "application/json",
      payload: JSON.stringify(visionPayload),
      muteHttpExceptions: true
    });
    const visionJson = JSON.parse(visionRes.getContentText());
    let textDetected = visionJson.responses[0]?.fullTextAnnotation?.text || "";

    // Cari NAMA dan MyKad/Passport (reg-ex Malaysia)
    let name = "";
    let idnum = "";

    // === REGEX UNTUK CARI NAMA & MYKAD ===
    // Cari line yang kemungkinan besar nama (selalunya atas perkataan "NAMA", "NAMA :", "NAME")
    let lines = textDetected.split('\n');
    for (let i = 0; i < lines.length; i++) {
      let ln = lines[i].trim();
      // Cari MyKad/Passport Number
      let idMatch = ln.match(/^(\d{6}-?\d{2}-?\d{4,6})$/) || ln.match(/^[A-Z]{1,2}\d{6,9}$/);
      if (idMatch && !idnum) idnum = idMatch[1] || idMatch[0];

      // Cari NAMA (selepas perkataan "NAMA" atau "NAME")
      if (
        (ln.match(/^NAMA\s*[:\-]?\s*(.+)/i) || ln.match(/^NAME\s*[:\-]?\s*(.+)/i)) &&
        !name
      ) {
        name = (ln.match(/^NAMA\s*[:\-]?\s*(.+)/i) || ln.match(/^NAME\s*[:\-]?\s*(.+)/i))[1].trim();
      }
      // Atau line selepas "NAMA"/"NAME"
      else if (
        (ln.toUpperCase() === "NAMA" || ln.toUpperCase() === "NAME") &&
        i + 1 < lines.length && !name
      ) {
        name = lines[i + 1].trim();
      }
    }

    // Fallback: Jika tak jumpa MyKad, cuba cari pattern 12 digit
    if (!idnum) {
      let fallback = textDetected.match(/(\d{6}-?\d{2}-?\d{4})/);
      if (fallback) idnum = fallback[1];
    }

    // Return JSON result ke frontend
    return ContentService.createTextOutput(
      JSON.stringify({ name: name, idnum: idnum, raw: textDetected })
    )
    .setMimeType(ContentService.MimeType.JSON)
    .setHeaders(corsHeaders);

  } catch (err) {
    // Return error ke frontend
    return ContentService.createTextOutput(
      JSON.stringify({ error: true, message: err.message || err })
    )
    .setMimeType(ContentService.MimeType.JSON)
    .setHeaders(corsHeaders);
  }
}

// ** CORS OPTIONS handler **
function doOptions(e) {
  return ContentService.createTextOutput("")
    .setMimeType(ContentService.MimeType.JSON)
    .setHeaders({
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type'
    });
}
