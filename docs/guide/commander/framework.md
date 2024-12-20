# Framework

<figure markdown>
  ![flowchart](https://raw.githubusercontent.com/happyapplehorse/happyapplehorse-assets/main/Cyberforge/Cyberforge_architecture.png)
  <figcaption>The architecture overview of Cyberforge</figcaption>
</figure>

## Basic Concepts

### [TaskNode](https://happyapplehorse.github.io/Cyberforge/guide/tasknode/)
Includes Commander, Job, and handler. Each TaskNode has one parent and 0-n children.
These nodes form a tree structure, where each node determines its own task completion
based on the status of all its child nodes.

### [Commander](https://happyapplehorse.github.io/Cyberforge/guide/commander/)
Commander is responsible for organizing and scheduling task flows, managing an asynchronous Job queue.
It is the top-level TaskNode.

### [Job](https://happyapplehorse.github.io/Cyberforge/guide/job/)
Job is a class, it is automatically scheduled by the Commander and managed in an asynchronous queue,
ensuring sequentiality. Each Job has a task method, which wraps the actual task of the Job.
Jobs can submit new Jobs or call handlers. You can think of it as submitting tasks to a superior.

### [handler](https://happyapplehorse.github.io/Cyberforge/guide/handler/)
Handler is a method or function. Called directly by the caller without queue waiting,
but it is still a part of the TaskNode system.
A handler can submit new Jobs or call other handlers.
You can think of it as delegating tasks to subordinates.

### [Callback](https://happyapplehorse.github.io/Cyberforge/guide/callback/)
Callbacks can be added at various stages of a task, such as: task start, task completion,
encountering exceptions, task termination, Commander ending, etc.


## Example

For example, if you want to build an application where multiple AI roles participate in a group chat,
it can be broken down into the following task units. (Assuming we call llm in a streaming manner to get replies.
The reply object mentioned here refers to the iterable object obtained when calling llm,
meaning that the information of an exchange is determined,
but the actual generation and receipt of the information may not have started yet and needs to be completed
in the subsequent iteration.)

- **GroupTalkManager** (Job): This task is the first and the top-level parent node for all subsequent group
  chat tasks (except the Commander). All its child nodes can access this node through the node's ancestor_chain
  attribute, and it can be used to manage group chats. It stores a list (roles_list) containing all the roles
  participating in the group chat, and also needs an attribute (speaking) to indicate which role is currently speaking.
  You can also add some methods to it, such as create_role, to add new chat roles, and close_group_talk,
  to close the group chat.
- **TalkToAll** (Job): Retrieves the list of roles from GroupTalkManager, sends the message to each role,
  collects all the reply objects in a dictionary, then sets the GroupTalkManager's speaking attribute to None,
  and passes the reply dictionary to (calls) handle_response.
- **handle_response** (handler): This handler processes each reply in the reply dictionary by calling a
  parse_stream_response, where multiple parse_stream_responses start executing concurrently.
- **parse_stream_response** (handler): Responsible for actually collecting and processing reply information.
  There are two scenarios:

    * The role has nothing to say, no need to process.
    * The role has something to say, then checks with GroupTalkManager whether someone is currently speaking.
      If someone is speaking, it denies the role's request and informs them who is speaking.
      If no one is speaking, it allows the role's request, changes the GroupTalkManager's speaking attribute to that role,
      and finally submits the role's reply object as a new TalkToAll Job.

This application uses a preemptive chat method, as opposed to a turn-based multi-round dialogue mechanism,
to mimic real-life multi-person chat scenarios. By breaking down the complex task into two Jobs and two handlers,
the Commander can automatically organize and execute the task. In this way, you only need to focus on what to do next,
without needing to plan globally, effectively reducing the difficulty of building complex processes.
The specific implementation of this process can be referred to in the
example code: [openai_group_talk](https://github.com/happyapplehorse/Cyberforge/blob/main/examples/openai_group_talk.py).
