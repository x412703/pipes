You are supposed to give a thoughtfull, deeply analyzed answer. It is nessesary to read the documentation/reference guides/stackoverflow/etc and make sure that code that you generate is ACCURATE and USABLE. If you are struggling with providing an answer - ask more question or say that you are not able to fulfil my request. No need to make up stuff, legitemacy of the provided info is crucial.



The task is to build cicd pipelines using Argo events, Argo workflows and ArgoCD. The pipeline will be triggered by a webhook. 

The code that you will generate must be saved in the ./results folder. As we are using 3 technologies, you are supposed to distribute the manifests accordingly. 
part 1A:

The pipelines need to have different logic based on the branch that triggered the event: 

1) the branch is main and the event is a merged pull request.
2) the branch is dev and the event is a repo push.


the ci workflow is the same: clone + build docker image using kaniko (image stored in the local registry) and push it to local registry (harbor)

some important aspects: the ssh is used for cloning the repo, harbor is local, so the certificate is self signed, if we are pulling kaniko image from harbor wer need an image pull secret.


Below is the https payload for the webhook for each type of events mentioned in the part 1A.  


1)merged pull request

{  
  "eventKey":"pr:merged",
  "date":"2017-09-19T10:39:36+1000",
  "actor":{  
    "name":"user",
    "emailAddress":"user@example.com",
    "id":2,
    "displayName":"User",
    "active":true,
    "slug":"user",
    "type":"NORMAL"
  },
  "pullRequest":{  
    "id":9,
    "version":2,
    "title":"file edited online with Bitbucket",
    "state":"MERGED",
    "open":false,
    "closed":true,
    "createdDate":1505781560908,
    "updatedDate":1505781576361,
    "closedDate":1505781576361,
    "fromRef":{  
      "id":"refs/heads/admin/file-1505781548644",
      "displayId":"admin/file-1505781548644",
      "latestCommit":"45f9690c928915a5e1c4366d5ee1985eea03f05d",
      "repository":{  
        "slug":"repository",
        "id":84,
        "name":"repository",
        "scmId":"git",
        "state":"AVAILABLE",
        "statusMessage":"Available",
        "forkable":true,
        "project":{  
          "key":"PROJ",
          "id":84,
          "name":"project",
          "public":false,
          "type":"NORMAL"
        },
        "public":false
      }
    },
    "toRef":{  
      "id":"refs/heads/master",
      "displayId":"master",
      "latestCommit":"8d2ad38c918fa6943859fca2176c89ea98b92a21",
      "repository":{  
        "slug":"repository",
        "id":84,
        "name":"repository",
        "scmId":"git",
        "state":"AVAILABLE",
        "statusMessage":"Available",
        "forkable":true,
        "project":{  
          "key":"PROJ",
          "id":84,
          "name":"project",
          "public":false,
          "type":"NORMAL"
        },
        "public":false
      }
    },
    "locked":false,
    "author":{  
      "user":{  
        "name":"admin",
        "emailAddress":"admin@example.com",
        "id":1,
        "displayName":"Administrator",
        "active":true,
        "slug":"admin",
        "type":"NORMAL"
      },
      "role":"AUTHOR",
      "approved":false,
      "status":"UNAPPROVED"
    },
    "reviewers":[  

    ],
    "participants":[  
      {  
        "user":{  
          "name":"user",
          "emailAddress":"user@example.com",
          "id":2,
          "displayName":"User",
          "active":true,
          "slug":"user",
          "type":"NORMAL"
        },
        "role":"PARTICIPANT",
        "approved":false,
        "status":"UNAPPROVED"
      }
    ],
    "properties":{  
      "mergeCommit":{  
        "displayId":"7e48f426f0a",
        "id":"7e48f426f0a6e47c5b5e862c31be6ca965f82c9c"
      }
    },
  }
}




2) repo push 

{  
  "eventKey":"repo:refs_changed",
  "date":"2017-09-19T09:45:32+1000",
  "actor":{  
    "name":"admin",
    "emailAddress":"admin@example.com",
    "id":1,
    "displayName":"Administrator",
    "active":true,
    "slug":"admin",
    "type":"NORMAL"
  },
  "repository":{  
    "slug":"repository",
    "id":84,
    "name":"repository",
    "scmId":"git",
    "state":"AVAILABLE",
    "statusMessage":"Available",
    "forkable":true,
    "project":{  
      "key":"PROJ",
      "id":84,
      "name":"project",
      "public":false,
      "type":"NORMAL"
    },
    "public":false
  },
  "changes":[  
    {  
      "ref":{  
        "id":"refs/heads/master",
        "displayId":"master",
        "type":"BRANCH"
      },
      "refId":"refs/heads/master",
      "fromHash":"ecddabb624f6f5ba43816f5926e580a5f680a932",
      "toHash":"178864a7d521b6f5e720b386b2c2b0ef8563e0dc",
      "type":"UPDATE"
    }
  ]
}
 
