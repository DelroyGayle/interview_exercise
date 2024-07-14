
# Unibuddy Engineering Exercise - My answers regarding this exercise

## Part 1

The service fails to start - ```npm run start:dev``` -  Use the messages to fix the code, so that the service runs successfully

### Answer

The service failed because there was a missing 'return' statement in
*src/message/message.resolver.ts*.

I added the word 'return' to Line 137

```
  @Mutation(() => ChatMessage)
  @UseGuards(GqlAuthGuard)
  async likeConversationMessage(
    @Args('likeMessageDto') likeMessageDto: LikeMessageDto,
    @AuthenticatedUser() authenticatedUser: IAuthenticatedUser,
  ): Promise<ChatMessage> {
    /*
      Bug fix regarding Part 1
      This line was missing a 'return' statement
      Fix: Add the word 'return'
    */
    return await this.messageLogic.like(likeMessageDto, authenticatedUser);
  }
```

## Part 2

A test is failing - ```npm run test``` - implement the code necessary to pass the test

### Answer

Tests failed because there was no code in place to actually update a record as *marked deleted*.

I observed in *src/message/message.data.ts* how other records were updated after any changes; therefore,<br>
I amended function *async delete(messageId: ObjectID)* accordingly,<br>the principal point, being to 
update *updateProperty* with the value *{ deleted: true }*

```
  async delete(messageId: ObjectID): Promise<ChatMessage> {
    /* 
       Regarding Part 2
       Allow a message to be marked as deleted
       Update the 'delete' property to 'true'
    */
   
      const query = { _id: messageId };
      const updateProperty = { deleted: true };
      const messageDeleted = await this.chatMessageModel.findOneAndUpdate(
        query,
        updateProperty,
        {
          new: true,
          returnOriginal: false,
        },
      );
      if (!messageDeleted) throw new Error('The message to be deleted does not exist');
      return chatMessageToObject(messageDeleted);
  }
  ```
![image](https://github.com/user-attachments/assets/109ae26a-b256-4ad4-833b-4c208f1f3dae)


### Please note:

After the service was started using *npm run start:dev*<br>
 and the 272 tests were performed after using *npm run test* 
 the following change was automatically made:

The following lines were added to **src/schema.gql**
```
type ChatConversation {
  id: ID!
  unreadMessageCount: Int
  lastMessage: ChatMessage
  pinnedMessages: [ChatMessage!]!
  pinnedMessagesCount: Int!
}
```

## Part 3 

### How would you go about implementing the solution?


I propose the usage of hashtags. That is, the user can add hashtags to the text of their message<br> and the Unibuddy Chat service would subsequently scan the message for such hashtags<br> and convert them to the chat service tags that are currently used internally.

As per the common usage of hashtags in the tech industry the following constraints regarding hashtags apply:
* Use only numbers and letters and the underscore character
* No spaces
* Cannot start with a number e.g.  #123 or  #123no
* A hashtag length would be up to 140 characters


So, for an example, here is a sample message:


> Greetings

> Looking forward to the ```#England``` vs ```#Spain``` game

> If interested we will be meeting up at the ```#PrinceRegent``` pub tonight.

> Please pass this message on.
> Thanks
> ```#Euro2024```



On receiving such a message the Unibuddy Chat service would retrieve the tags:
```england spain princeregent euro2024```<br>
Then the functionality would convert them to lowercase and interpret them as Chat tags for the Conversation tags.<br>
The tags attached to the *ConversationChatModel* will in turn be updated with any tags found in the *MessageChatModel*. 

When the user amends any message within a conversation with new or amended hashtags,<br>
the chat service will update the Message Tags accordingly;<br> which in turns means that the Conversation Tags would have to be updated accordingly.

If the user deletes hashtags from a message the chat service would in like manner remove the corresponding tags.<br>
Functionality needs to be put in place to mark a tag<br> as deleted so that the Unibuddy Chat service would know to remove the corresponding Conversation Tag.

#### Finding Messages

In order for users to find messages based on these tags,<br>
Method 1 - a text solution<br>
I propose that the user could create a message with a sole line consisting of the word ```#find``` followed by the tags in question e.g.<br>
```#find euro2024 england spain```

Method 2 - UI solution<br>
Alternatively, depending on the current implementation, a search bar could be implemented so that the user could enter the search tags.


Either way, once the search is commenced, then the chat service would retrieve messages which have the tags<br>
```euro2024 OR england OR spain```


### What problems might you encounter?


The main problem I see with this present approach is figuring out a uniform, consistent method for adding tags to the message model<br> so that they can be converted to conversation chat tags as easily as possible.<br>
This would imply the use of *two* distinct model types with *two* different types of tags.<br>
Thus, the primary challenges are how to efficiently and reliably get both models<br> to cooperate while minimizing complexity.


#### My MockUp 

**Modify src/schema.gql**

```
type ChatMessage {
  id: ID!
  text: String!
  created: DateTime!
  sender: UserField!
  deleted: Boolean!
  resolved: Boolean!
  likes: [ObjectId!]!
  likesCount: Int!
  richContent: RichMessageContent
  reactions: [Reaction!]
  isSenderBlocked: Boolean
  **hashtags: [String!]**
}
```

**Modify src/message/models/message.entity.ts**


@ObjectType()
@Directive('@key(fields: "id")')
export class ChatMessage { …


  @Field(() => [String], { nullable: true })
  **hashtags?: String[];**
}


export class SocketChatMessage { …
**hashtags?: String[];**
}


**Modify src/message/models/message.model.ts**


  @Prop()
  **hashtags: Array<string>;**


**Modify src/message/message.logic.ts**


```
   const MAXTAGSLENGTH = 140;
   let newTags;
   const matches = text.match(/#([A-ZAa-z_]\w*)/g);
    if (matches) {
	      matches.forEach(word  => newTags.push(word.slice(1)
                                                  .substring(0, MAXTAGSLENGTH).toLowerCase())); 
                 // Combine pre-existing message tags with new ones - remove duplicates
                 newTags = Array.from(new Set(message.hashtags.slice(), newTags));
    } else {
	      newTags = message.hashtags;
   }
   
    const sendMessageEvent = new SendMessageEvent({
      id: message.id,
      text: this.safeguardingService.clean(message.text),
      created: message.created,
      sender: {
        id: sender.id,
        firstName: sender.firstName,
        profilePhoto: sender.profilePhoto,
        accountRole,
      },
      deleted: message.deleted,
      likes: message.likes,
      likesCount: message.likesCount,
      richContent: await this.mapRichContent(messageDto, message),
      resolved: message.resolved,
      isSenderBlocked: false,
      // The hashtags array is added here
      hashtags: newTags,
    });
``` 

   
### How would you go about testing?


Following the code shown in **src/message/message.logic.spec.ts**<br>
Here is a sample Jest test:
 
 ```
  describe('create', () => {
    it('can create a new message with hashtags', async () => {
      jest.spyOn(messageData, 'create');
      await messageLogic.create(
        { text: '#Hello #World. This is my #hashtag message text', conversationId },
        { ...validUser, userId: senderId },
      );


   expect(messageData.hashtags).toContain('hello', 'world', 'hashtag');
});
```



### What you might do differently


Because tags are currently attached to Conversations it means that additional kind of tagging is needed for messages i.e. hashtags. 
<br>
However that means that tagging functionality is currently in two separate locations.<br>Thus, I would modify the message models and messaging functionality to allow tags to be attached only to messages<br>rather than conversations in order to make the implementation of tagging capability easier.<br>In this manner, future developments will only require changes to one source of functionality.

