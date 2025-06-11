const VISION_API_KEY = "AIzaSyBesfmexK3feou_oP2J-b3lD3PAbQUwqNA"; // Gantikan dengan API key anda

function doPost(e) {
  try {
    const corsHeaders = {
      'Access-Control-Allow-Origin': '*',
      'Access-Control-Allow-Methods': 'POST, OPTIONS',
      'Access-Control-Allow-Headers': 'Content-Type'
    };

    const data = JSON.parse(e.postData.contents);
    const imageBase64 = data.image;

    const visionPayload = {
      requests: [
        {
          image: { content: imageBase64 },
          features: [{ type: "TEXT_DETECTION" }]
        }
      ]
    };

    const response = UrlFetchApp.fetch(
      `https://vision.googleapis.com/v1/images:annotate?key=${VISION_API_KEY}`,
      {
        method: "post",
        contentType: "application/json",
        payload: JSON.stringify(visionPayload),
        muteHttpExceptions: true
      }
    );

    const result = JSON.parse(response.getContentText());
    const text = result.responses?.[0]?.fullTextAnnotation?.text || "";

    let name = "";
    let idnum = "";

    const lines = text.split('\n');
    for (let i = 0; i < lines.length; i++) {
      const line = lines[i].trim();

      const idMatch = line.match(/^(\d{6}-?\d{2}-?\d{4,6})$/) || line.match(/^[A-Z]{1,2}\d{6,9}$/);
      if (idMatch && !idnum) idnum = idMatch[1] || idMatch[0];

      const nameMatch = line.match(/^NAMA\s*[:\-]?\s*(.+)/i) || line.match(/^NAME\s*[:\-]?\s*(.+)/i);
      if (nameMatch && !name) {
        name = nameMatch[1].trim();
      } else if (
        (line.toUpperCase() === "NAMA" || line.toUpperCase() === "NAME") &&
        i + 1 < lines.length && !name
      ) {
        name = lines[i + 1].trim();
      }
    }

    if (!idnum) {
      const fallback = text.match(/(\d{6}-?\d{2}-?\d{4})/);
      if (fallback) idnum = fallback[1];
    }

    const output = {
      name: name,
      idnum: idnum,
      raw: text
    };

    return ContentService.createTextOutput(JSON.stringify(output)).setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService.createTextOutput(JSON.stringify({ error: true, message: err.message })).setMimeType(ContentService.MimeType.JSON);
  }
}

// Untuk respon permintaan CORS dari browser (OPTIONS)
function doOptions(e) {
  const response = HtmlService.createHtmlOutput("");
  const headers = response.getResponse().getHeaders();
  headers["Access-Control-Allow-Origin"] = "*";
  headers["Access-Control-Allow-Methods"] = "POST, OPTIONS";
  headers["Access-Control-Allow-Headers"] = "Content-Type";
  return response;
}
