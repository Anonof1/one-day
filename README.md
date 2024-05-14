using System.Collections;
using TMPro;
using UnityEngine;
using UnityEngine.SceneManagement;
using UnityEngine.UI;
using DG.Tweening;

public class DialogueManage : MonoBehaviour
{ 
    public TextMeshProUGUI personName;
    public TextMeshProUGUI dialogue;
    public MicrophoneManager micmanage;
    public ApiManager API;
    public GameObject dialogueBox;
    public Image BG;
    
    public GameObject questBox;
    public GameObject player;
    public GameObject fixedCam;
    public static int sentenceIndex = -1;
    public StoryScene currentScene;
    public State state = State.COMPLETED;
    public bool voice = false;
    public string tmpText;
    public bool canSkip = false;

    

    public void PlayScene(StoryScene scene)
    {
        currentScene = scene;
        sentenceIndex = -1;
        PlayNextSentence();
    }
    public void PlayNextSentence()
    {
        MicInactive();
        ShowEventBG();
        ShowPlayer();
        ShowHint();
        GameManage.isEvent = true;
        // check switch scene
        if (currentScene.Sentences[sentenceIndex + 1].switchScene)
        {
            GameManage.score += GameManage.scoreTemp;
            GameManage.scoreTemp = 0;
            SceneManager.LoadScene(sceneBuildIndex: SceneManager.GetActiveScene().buildIndex+1);
        }
        // check if dialogue box is active
        if (currentScene.Sentences[sentenceIndex + 1].isInactive)
        {
            dialogueBox.SetActive(false);
            MicActive();
        }
        // if dialogue box is active
        else
        {
            if (currentScene.Sentences[sentenceIndex + 1].apiVoice)
            {
                MicActive();
            }
            if (currentScene.Sentences[sentenceIndex + 1].apiText)
            {
                API.sendAPI('T',currentScene.Sentences[++sentenceIndex].text);
            }
            else
            {
                StartCoroutine(TypeText(currentScene.Sentences[++sentenceIndex].text.Replace("%","\n")));
            }

            personName.text = currentScene.Sentences[sentenceIndex].speaker.speakerName == "Player" ? PlayerPrefs.GetString("PlayerName","???") : currentScene.Sentences[sentenceIndex].speaker.speakerName;
            personName.color = currentScene.Sentences[sentenceIndex].speaker.textColor;
            
        }

        if (currentScene.Sentences[sentenceIndex].Animation != "")
        {
            
        }
        
    }

    public void EventStart()
    {
        dialogueBox.SetActive(true);
        sentenceIndex += 1;
        PlayNextSentence();
    }

    private void ShowEventBG()
    {
        if (currentScene.Sentences[sentenceIndex + 1].showBG)
        {
            BG.sprite = currentScene.background;
            BG.DOFade(1f, 3f);
        }
        else
        {
            BG.DOFade(0f, 3f);
        }
    }
    private void ShowHint()
    {
        if (currentScene.Sentences[sentenceIndex+1].hint != "")
        {
            questBox.SetActive(true);
            QuestManage.Show(currentScene.Sentences[sentenceIndex+1].hint.Replace("%","\n"));
        }
        else
        {
            questBox.SetActive(false);
        }
    }

    private void ShowPlayer()
    {
        player.SetActive(currentScene.Sentences[sentenceIndex + 1].showPlayer);
        fixedCam.SetActive(!currentScene.Sentences[sentenceIndex + 1].showPlayer);
    }


    public enum State
    {
        PLAYING,COMPLETED
    }
    public IEnumerator TypeText(string text)
    {
        tmpText = text;
        dialogue.text = "";
        state = State.PLAYING;
        canSkip = false;
        int wordIndex = 0;
        if (text.Length > 0)
        {
            while (state != State.COMPLETED)
            {
                dialogue.text += text[wordIndex];

                yield return new WaitForSeconds(0.05f);
                if (++wordIndex == text.Length)
                {
                    state = State.COMPLETED;
                    break;
                }

                if (wordIndex>10)
                {
                    canSkip = true;
                }
            }
        }
        
    }
    public bool IsInactive()
    {
        return currentScene.Sentences[sentenceIndex+1].isInactive;
    }

    public bool IsCompleted()
    {
        return state == State.COMPLETED;
    }

    public bool IsLastSentence()
    {
        return sentenceIndex + 1 == currentScene.Sentences.Count;
    }

    public bool IsNextScene()
    {
        return currentScene.nextScene;
    }

    public void MicActive()
    {
        micmanage.setActive();
        voice = true;
    }
    public void MicInactive()
    {
        micmanage.setInactive();
        voice = false;
    }



}
