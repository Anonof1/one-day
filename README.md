using System;
using System.Collections;
using System.Collections.Generic;
using System.IO;
using System.Text;
using Unity.Mathematics;
using UnityEngine;
using UnityEngine.Networking;

public class RespondJson //
{
    public string bot_id;
    public string intent;
    public List_Obj[] messages;
    public string source;
}
[System.Serializable]
public class List_Obj
{
    public string audio_url;
    public string mid;
    public string text;
    public string timestamp;
    public string type;
}

public class ApiManager : MonoBehaviour //
{
    private string api_url = "https://asia-southeast1-botnoiasr.cloudfunctions.net/voicebotBU";
    public GameObject chatBubble;
    public DialogueManage dialogueManage;
    public static string displayText;
    public static string hintText;
    private byte[] fileContents = null;
    private string responseVoice;
    private string responseText;
    public AudioSource audioSource;
    public AudioClip myAudio;
    public static bool waiting;
    public static bool speaking;
    private string apiTextName;
    private static List<string> speakerID = new List<string>
    {
        "เอวา", "โบ", "คุณงาม", "แม็กซ์", "อลัน", "ไซเรน", "อลิสา", "เลโอ", 
        "นาเดียร์", "วนิลา", "อนันดา", "ไอลีน", "ฮิโระ", "ครูดีดี๊", "เจ้าเนิร์ด", 
        "โอโตะ", "อาวอร์ม", "ปีเตอร์", "คริส", "เจสัน", "หญิงไอโกะ", "น้าเกรซ", 
        "อาจารย์หลิน", "สโม๊ค", "โตโต้", "นายเบรด", "เนโอ", "สาลี", "แบมบู", 
        "ผู้ใหญ่ลี", "เท็ดดี้", "โนรา", "แม็ท", "เอลลี่", "ข้าวตอก", "อ้ายเถิน", 
        "ทัช", "เบน", "คำแก้ว", "น้องโนว์", "ท็อป", "ครูอภิวัฒน์", "พลอย", 
        "แทน", "เอไอ", "ฟอมมี่", "มานี", "บี", "กิ่ง", "ธนัท", "นิค", "ปรีดี", 
        "เหมียว", "ตี๋", "ro","boss"
    };

    public string speaker = "โบ";
    
    
    public void sendAPI(char type, string text) //
    {
        waiting = true;
        Debug.Log("sendAPI");
        StartCoroutine(wait(1f));
        switch (type)
        {
            case 'V':
                StartCoroutine(sendVoiceRequestAPI());
                break;
            case 'T':
                StartCoroutine(sendTextRequestAPI(text));
                break;
        }
    }
    
    private IEnumerator sendVoiceRequestAPI() //
    {
        //
        string fileName = "Record.wav"; 
        string filePath = Path.Combine(Application.streamingAssetsPath + "/Recordings/", fileName);

        //
        if (!File.Exists(filePath))
        {
            Debug.LogError($"File {filePath} not found!");
        }

        // Read file in binary mode
        using (FileStream fileStream = new FileStream(filePath, FileMode.Open, FileAccess.Read))
        {
            // -----------------------------------------------------------------
            fileContents = new byte[fileStream.Length];
            fileStream.Read(fileContents, 0, (int)fileStream.Length);
        }
        // -----------------------------------------------------------------
        Debug.Log($"File {filePath} contains {fileContents.Length} bytes.");
        string hexString = BitConverter.ToString(fileContents);

        //
        string post_to_api = "{\"customerId\": \"" + SystemInfo.deviceUniqueIdentifier+PlayerPrefs.GetInt("ID",0) + "\", " +
            "\"botId\": \"64fdb6041016547581b119dc\", " +
            "\"input\": {\"type\": \"audio\", \"object\": {" +
            "\"language\": \"th\", " +
            "\"audioData\": \"" + hexString + "\", " +
            "\"dataFormat\": \"hex\", " +
            "\"audioFormat\": \"wav\"}}, " +
            "\"output\": {\"type\": \"audio\", " +
            "\"object\": {\"speakerId\": \"" + speaker + "\"}}}";
        

        Debug.Log("Sending API request");

        //
        UnityWebRequest request = UnityWebRequest.PostWwwForm(api_url, post_to_api);
        byte[] bodyRaw = Encoding.UTF8.GetBytes(post_to_api);
        request.uploadHandler = new UploadHandlerRaw(bodyRaw);
        request.downloadHandler = new DownloadHandlerBuffer();
        request.SetRequestHeader("Content-Type", "application/json");

        //
        yield return request.SendWebRequest();
        //waiting = false;

        if (request.result == UnityWebRequest.Result.Success) //
        {
            Debug.Log("API response received: " + request.downloadHandler.text);
            responseVoice = request.downloadHandler.text;
            if (responseVoice.Contains("fail"))
            {
                waiting = false;
                DialogueManage.sentenceIndex -= 1;
                dialogueManage.PlayNextSentence();
                yield break;
            }
            StartCoroutine(handleVoiceJSON());
        }
        else //
        {
            Debug.LogError("API request failed: " + request.error);
            waiting = false;
            DialogueManage.sentenceIndex -= 1;
            dialogueManage.PlayNextSentence();
        }
    }
    
    private IEnumerator sendTextRequestAPI(string input) //
    {
        string filePath = Path.Combine(Application.streamingAssetsPath + "/Recordings/", input+".wav");
        if (File.Exists(filePath))
        {
            string post_to_api_textonly = "{\"customerId\": \"" + SystemInfo.deviceUniqueIdentifier+PlayerPrefs.GetInt("ID",0)+ "\", " +
                                          "\"botId\": \"64fdb6041016547581b119dc\", " +
                                          "\"input\": {\"type\": \"text\", \"object\": {" +
                                          "\"text\": \"" + input + "\"}}, " +
                                          "\"output\": {\"type\": \"text\"}}";
            UnityWebRequest requestT = UnityWebRequest.PostWwwForm(api_url, post_to_api_textonly);
            byte[] bodyRawT = Encoding.UTF8.GetBytes(post_to_api_textonly);
            requestT.uploadHandler = new UploadHandlerRaw(bodyRawT);
            requestT.downloadHandler = new DownloadHandlerBuffer();
            requestT.SetRequestHeader("Content-Type", "application/json");

            //
            yield return requestT.SendWebRequest();

            if (requestT.result == UnityWebRequest.Result.Success) //
            {

                if (File.Exists(filePath))
                {
                    Debug.Log("Audio exist for " + filePath);

                    UnityWebRequest www = UnityWebRequestMultimedia.GetAudioClip(filePath, AudioType.WAV);
                    yield return www.SendWebRequest();
                    myAudio = DownloadHandlerAudioClip.GetContent(www);
                    RespondJson respond = new RespondJson();
                    respond = JsonUtility.FromJson<RespondJson>(requestT.downloadHandler.text);
                    speaking = true;
                    waiting = false;
                    StartCoroutine(dialogueManage.TypeText(respond.messages[0].text));
                    audioSource.PlayOneShot(myAudio, 0.6f);
                    yield return new WaitForSeconds(myAudio.length);
                    speaking = false;
                    if (dialogueManage.IsLastSentence() && !dialogueManage.IsNextScene())
                    {
                        //dialogueManage.MicActive();
                    }
                }
                else
                {
                    Debug.Log("API response received: " + requestT.downloadHandler.text);
                    responseVoice = requestT.downloadHandler.text;
                    StartCoroutine(handleVoiceJSON());
                }
            }
            else //
            {
                Debug.LogError("API request failed: " + requestT.error);
                waiting = false;
                DialogueManage.sentenceIndex -= 1;
                dialogueManage.PlayNextSentence();
            }
        }
        else
        {
            string post_to_api = "{\"customerId\": \"" + SystemInfo.deviceUniqueIdentifier+PlayerPrefs.GetInt("ID",0) + "\", " +
                                 "\"botId\": \"64fdb6041016547581b119dc\", " +
                                 "\"input\": {\"type\": \"text\", \"object\": {" +
                                 "\"text\": \"" + input + "\"}}, " +
                                 "\"output\": {\"type\": \"audio\", " +
                                 "\"object\": {\"speakerId\": \"" + speaker + "\"}}}";

            Debug.Log(post_to_api);

            //
            UnityWebRequest request = UnityWebRequest.PostWwwForm(api_url, post_to_api);
            byte[] bodyRaw = Encoding.UTF8.GetBytes(post_to_api);
            request.uploadHandler = new UploadHandlerRaw(bodyRaw);
            request.downloadHandler = new DownloadHandlerBuffer();
            request.SetRequestHeader("Content-Type", "application/json");

            //
            yield return request.SendWebRequest();
        
            if (request.result == UnityWebRequest.Result.Success) //
            {
                Debug.Log("API response received: " + request.downloadHandler.text);
                responseVoice = request.downloadHandler.text;
                StartCoroutine(handleVoiceJSON());
            }
            else //
            {
                Debug.LogError("API request failed: " + request.error);
                waiting = false;
                DialogueManage.sentenceIndex -= 1;
                dialogueManage.PlayNextSentence();
            }
        }
    }

    public IEnumerator handleVoiceJSON()
    {
        RespondJson respondVJson = new RespondJson();
        try
        {
            respondVJson = JsonUtility.FromJson<RespondJson>(responseVoice);
            StartCoroutine(receiveAudioResponse(respondVJson));
        }
        catch (Exception e)
        {
            Debug.Log("Exception = " + e);
            yield break;
        }
        yield return respondVJson;
    }

    public IEnumerator receiveAudioResponse(RespondJson respondJson) //
    {
        //Debug.Log(respondJson.messages.Length);
        //download and play every audio clip in order
        for (int i = 0; i < respondJson.messages.Length; i++)
        {
            // check if the received message has the key
            if (respondJson.messages[i].text.Contains("$"))
            {
                string temp = respondJson.messages[i].text.Remove(0,1);
                DialogueManage.sentenceIndex = Convert.ToInt32(temp)-2;
                //Debug.Log(dialogueManage.sentenceIndex);
                /*var startIndex = respondJson.messages[i].text.IndexOf("(", StringComparison.Ordinal);
                var endIndex = respondJson.messages[i].text.IndexOf(")", StringComparison.Ordinal);
                if (startIndex != -1 && endIndex != -1 && endIndex > startIndex)
                    // Extract the substring between the starting and ending characters
                {
                    Debug.Log(respondJson.messages[i].text.Substring(startIndex + 1, endIndex - startIndex - 1));
                    StartCoroutine(sendTextRequestAPI(respondJson.messages[i].text.Substring(startIndex + 1, endIndex - startIndex - 1)));
                }*/
            }
            else if (respondJson.messages[i].text=="+")
            {
                GameManage.scoreTemp++;
                //Debug.Log("TempScore = "+GameManage.score);
            }
            else if (respondJson.messages[i].text=="-")
            {
                GameManage.scoreTemp++;
                //Debug.Log("TempScore = "+GameManage.score);
            }
            else
            {
                string audio_url = respondJson.messages[i].audio_url;
                UnityWebRequest www = UnityWebRequestMultimedia.GetAudioClip(audio_url, AudioType.WAV);
                yield return www.SendWebRequest();
                myAudio = DownloadHandlerAudioClip.GetContent(www);
                
                if (dialogueManage.currentScene.Sentences[DialogueManage.sentenceIndex].text == respondJson.intent)
                {
                    SaveWav.Save(respondJson.intent, myAudio);
                    Debug.Log("Saved audio: "+ respondJson.intent+".wav");
                }
                
                //displayText = respondJson.messages[i].text;
                speaking = true;
                waiting = false;
                StartCoroutine(dialogueManage.TypeText(respondJson.messages[i].text
                    .Replace("%", "\n")
                    .Replace("ชั้น", "ฉัน")
                    .Replace("มั้ย", "ไหม")
                    .Replace("รึ", "หรือ")
                    .Replace("แคนด\u0e35\u0e49 ทาวเวอร\u0e4c", "Candy Tower")
                    .Replace("โก\u0e4aส ค\u0e34ลเลอร\u0e4c", "Ghost Killer")
                    .Replace("เฟ\u0e34ร\u0e4cส เล\u0e34ฟ", "First Love")));
            
                
                
                //play clip until finish then continue and remove chatBubble
                audioSource.PlayOneShot(myAudio,0.6f);
                //create chatBubble
                //var temp = Instantiate(chatBubble, transform.position, quaternion.identity,transform);
                //AudioSource.PlayClipAtPoint(myAudio, transform.position, 1);
                yield return new WaitForSeconds(myAudio.length);
                speaking = false;
                //Destroy(temp);
                if (dialogueManage.IsLastSentence()&&!dialogueManage.IsNextScene())
                {
                    dialogueManage.MicActive();
                }
            }
        }
    }

    public IEnumerator wait(float time)
    {
        dialogueManage.personName.text = "โมกุ ไอ";
        dialogueManage.personName.color = Color.magenta;
        dialogueManage.dialogue.text = "";
        displayText = "";
        var temp = Instantiate(chatBubble, transform.position, quaternion.identity,transform);
        while (waiting)
        {
            if (waiting)
            {
                dialogueManage.dialogue.text += ". ";
                displayText += ". ";
                yield return new WaitForSeconds(time);
            }
            if (waiting)
            {
                dialogueManage.dialogue.text += ". ";
                displayText += ". ";
                yield return new WaitForSeconds(time);
            }
            if (waiting)
            {
                dialogueManage.dialogue.text += ". ";
                displayText = ". ";
                yield return new WaitForSeconds(time);
            }
        }
        Destroy(temp);
    }
}
