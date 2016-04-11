## example server
### echo server on GAE

``` go
package linebotapi_gae

import (
    "linebotapi"

    "net/http"

    "google.golang.org/appengine"
    "google.golang.org/appengine/urlfetch"
)

func init() {
    http.HandleFunc("/callback", handler)
}

func handler(w http.ResponseWriter, r *http.Request) {
    c := appengine.NewContext(r)

    result, err := linebotapi.ParseRequest(r.Body)
    if err != nil {
        panic(err)
    }
    client := http.Client{
        Transport: &urlfetch.Transport{
            Context: c,
        },
    }

    cred := linebotapi.Credential{
        ChannelId: ****,        // Your Channel ID
        ChannelSecret: "****",  // Your Channel Secret
        Mid: "****",            // Your MID
    }

    var messages = make(map[string][]linebotapi.MessageContent)
    for _, m := range result.Result {
        _, exists := messages[m.Content.From]
        if !exists {
            messages[m.Content.From] = make([]linebotapi.MessageContent, 0)
        }
        if m.Content.ContentType == linebotapi.ContentTypeText {
            // Text
            messages[m.Content.From] = append(messages[m.Content.From], linebotapi.MessageContent{
                ContentType: linebotapi.ContentTypeText,
                ToType: linebotapi.ToTypeUser,
                Text: m.Content.Text,
            });
        }
        if m.Content.ContentType == linebotapi.ContentTypeSticker {
            // Sticker(preinstall sticker only?)
            messages[m.Content.From] = append(messages[m.Content.From], linebotapi.MessageContent{
                ContentType: linebotapi.ContentTypeSticker,
                ToType: linebotapi.ToTypeUser,
                ContentMetadata: map[string]string{
                    "STKID": m.Content.ContentMetadata["STKID"],
                    "STKPKGID": m.Content.ContentMetadata["STKPKGID"],
                },
            });
        }
    }
    for k := range messages {
        contacts, err := linebotapi.GetUserProfiles(client, cred, []string{k})
        if err != nil {
            panic(err)
        }
        messages[k] = append(messages[k], linebotapi.MessageContent{
            ContentType: linebotapi.ContentTypeText,
            ToType: linebotapi.ToTypeUser,
            Text: contacts.Contacts[0].DisplayName + " さん",
        });
        err = linebotapi.SendMessages(client, cred, []string{k}, messages[k], 0)
        if err != nil {
            panic(err)
        }
    }
}


```