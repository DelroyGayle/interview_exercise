
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
I amended function *async delete(messageId: ObjectID)* accordingly, the principal point, being to 
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

