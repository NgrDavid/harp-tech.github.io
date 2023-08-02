# Bonsai.Harp

All [Harp Devices](https://harp-tech.org/Devices/device_list.html) implement the [Harp Protocol](https://harp-tech.org/About/How-HARP-works/index.html) via which they can communicate with an host PC. The `Bonsai.Harp` library provides an implementation of the Harp protocol that can be used to interface with any Harp device.

The `Bonsai.Harp` library provides the following operators:

![Bonsai.Harp Operators](~/images/core-operators.svg)

It is crucial to understand the function of each of these operators as they are the building blocks of any Harp application. Additionally, when interfacing with a particular model of an Harp Device using Bonsai, device-specific variants of these operators are provided to expose a device-specific, high-level interface.

## Device

The [`Device`](xref:Bonsai.Harp.Device) operator is the first node you will probably add to your workflow when using any Harp Device. This source is responsible for establishing a serial connection link with the device and providing an interface that can be used to send and receive messages, represented by [`HarpMessage`](xref:Bonsai.Harp.HarpMessage) objects. Messages sent from the host application into the node represent commands to be sent to the device, whereas message objects emitted by the device node represent command replies or events sent by the device to the host.

> [!Note]
> The `Device` node also provides a default editor that allows you to configure core device settings and upload new firmware to the board in the form of `.hex` files. To open this dialog you can simply double-click the node while bonsai is not running.

The current specification of the `Harp Protocol` defines three [`HarpMessage types`](xref:Bonsai.Harp.MessageType):

- `Read`: A command from the host to the device, that will result in a [`HarpMessage`](xref:Bonsai.Harp.HarpMessage) response, from the device, containing the value stored in the specified `register`. All `registers` are expected to allow read access, as by protocol specification.
- `Write`: A command issued by the host to the device to request the change of a particular `register`'s value. As per the protocol standard, if the `write` operation is returns successfully, the device should return a timestamped [`HarpMessage`](xref:Bonsai.Harp.HarpMessage) of the `Write` type with the newly set value.
- `Event`: The only message type that is emitted from the device to the host without a corresponding prior command. These messages are used to notify the host of events that occur on the device, such as a periodic computations (*e.g.* ADC reads), or other sporadic events (*e.g.* toggle on a digital input pin).

The following `Bonsai.Harp` operators provide the core functionality for generating [`HarpMessage`](xref:Bonsai.Harp.HarpMessage) objects (to issue commands), and for processing `HarpMessages` (to handle notifications).

## CreateMessage

In order to generate messages and thus be able to issue commands to the device two core operators are provided: [`CreateMessage`](xref:Bonsai.Harp.CreateMessage) and [`Format`](xref:Bonsai.Harp.Format). Both operators receive a single input and are able to generate a [`HarpMessage`](xref:Bonsai.Harp.HarpMessage) object. However, they differ on how the message is generated. [`CreateMessage`](xref:Bonsai.Harp.CreateMessage) will fully define a [`HarpMessage`](xref:Bonsai.Harp.HarpMessage) using the settings available in the property grid, whereas [`Format`](xref:Bonsai.Harp.Format) will use the input sequence to generate a [`HarpMessage`](xref:Bonsai.Harp.HarpMessage)'s payload. Conceptually, the latter operator is similar to [`Format`](xref:Bonsai.Harp.Format) methods present in several programming languages, such as [`String.Format`](https://learn.microsoft.com/en-us/dotnet/api/system.string.format?view=net-7.0) in `C#`.

Take as example the following hypothetical [`HarpMessage`](xref:Bonsai.Harp.HarpMessage):

```
Address: 32
MessageType: Write
Length: 1
PayloadType: U16
Payload: 10
```

To generate it, one could use the [`CreateMessage`](xref:Bonsai.Harp.CreateMessage) operator:

:::workflow
![CreateMessage](~/workflows/create-message.bonsai)
:::

## Format

or, equivalently, the the following [`Format`](xref:Bonsai.Harp.Format) operator:

:::workflow
![FormatMessage](~/workflows/format.bonsai)
:::

As opposed to the previous dyad of operators, the following are used to process [`HarpMessage`](xref:Bonsai.Harp.HarpMessage) objects that are received from the device. These operators can be used to filter, parse, and transform incoming messages.

## FilterRegister

In its simplest form, [`FilterRegister`](xref:Bonsai.Harp.FilterRegister) functions as a [`Condition`](xref:Bonsai.Reactive.Condition) that checks, for each incoming message, whether the message's address matches the ones specified in the property grid  If the condition is met, the message is passed through the operator, otherwise it is discarded. It should be noted that the default filtering behavior can be inverted by modifying the `FilterType` property. *I.e.* "Allow every message that does **not** match to pass.

For instance, one could construct a filter to listen for the 1 Hertz heartbeat events from an `Harp Device` by:

:::workflow
![FilterRegister](~/workflows/filter-message.bonsai)
:::

## FilterMessageType

Sometimes it is also useful to filter message types, on top of the address. To match that need, an additional operator, `FilterMessageType`, is provided. This operator will filter messages based on their `MessageType` property and can be used in combination with the previous operator. For instance, if one would be interested in listening to the echo emitted from the device (*i.e.* a `Write` message) after the previous [`CreateMessage`](xref:Bonsai.Harp.CreateMessage) or [`Format`](xref:Bonsai.Harp.Format) operators, one could use the following [`FilterRegister`](xref:Bonsai.Harp.FilterRegister) operator:

:::workflow
![FilterMessageType](~/workflows/filter-messagetype-register.bonsai)
:::

## Parse

While [`FilterRegister`](xref:Bonsai.Harp.FilterRegister) can be used to filter incoming messages, the value of every single `HarpMessage` in the sequence will return unchanged. However, in many cases, one might be interested in extracting, and converting, the value of the message's payload. This is where the [`Parse`](xref:Bonsai.Harp.Parse) operator comes handy. This operator will extract and parse the payload of any incoming message and return a new sequence of parsed values. The type of the parsed values will depend on the `PayloadType` of the incoming message. For instance, if the `PayloadType` is `U16`, the operator will return a sequence of `ushort` values. The following workflow shows how to parse the payload of the previous example:

:::workflow
![Parse](~/workflows/parse.bonsai)
:::

It should be noted that if an [`Address`](xref:Bonsai.Harp.ParseMessagePayload.Address) is defined, the node will additionally filter the incoming sequence of messages prior to the parsing operation.

## High-level device interfaces

All the previous examples assume the users knows the `Address` they would like to target. However, as the number of register and different boards grow, it becomes increasingly difficult to keep track of what function each `Address` corresponds to. To address this issue, all the previous nodes were created as `Polymorphic operators`. In short, by modifying the `Payload` property from the `*MessagePayload` default to a specific register name, the operator will automatically refresh its properties, in order to match the `Payload` structure of the message for the selected `Address` name.

This *morphing* behavior:

- Affords the creation of `Harp Messages` of any type without previous knowledge of register specifications:
![whoami-read](~/images/whoami_read.png)

- For registers with `Enumerable`-like inputs, allows users easy access to available inputs via a drop-down menu:
![resetdevice-write](~/images/resetdevice-write.png)

- For registers with a complex structure, allows users to easily, and simultaneously, manipulate the different fields to compose a message:
![opcontrol-write](~/images/opcontrol-write.png)


The aforementioned high-level operators only expose the `Core` registers, common to all `Harp devices`. To access device-specific functionality, users will have to download Bonsai packages targeting specific `Harp devices`. Once installed, the packages will expose additional operators in the toolbox, identical, in name and syntax, to the others previously described here. These device-specific operators will show-up with a different namespace, and will be able to target device-specific registers (or `Application` registers). [Such an example can be found here.](TODO fill in with behavior)

## The Harp Device pattern

The previous sections covered the basic functionality of the `Bonsai.Harp` library. In theory, all the previous operators could function, in parallel, by branching/merging `HarpMessages` data streams from/to the `Harp Device`. For example:

:::workflow
![DevicePattern](~/workflows/device-pattern-nosubjects.bonsai)
:::

However, as workflows grow, this pattern will quickly become cumbersome to manage. To address this issue, we recommend using [Subjects](https://bonsai-rx.org/docs/articles/subjects.html) to interact with the Device.
As such, we start by adding a [`PublishSubject`](https://bonsai-rx.org/docs/articles/subjects.html#publishsubject) to the right side of the `Harp Device`. This subject, will allow us to receive the stream of `HarpMessages` in multiple places of our workflow, which eliminates the need for explicitly branching the original data stream:

:::workflow
![DevicePatternPublish](~/workflows/device-pattern-publish.bonsai)
:::

This pattern takes care of the output of the `Harp Device` (*i.e.* incoming messages). However, the same way that we might want *read* the stream of `Harp Messages` from multiple places in our workflow, we might also want to send (or *write*) `Commands` to the device from several parallel branches. To address this issue, we will add a second subject, but this time to the input of the `Device` node. This [`Source`](https://bonsai-rx.org/docs/articles/subjects.html#source-subjects) [`Behavior Subject`](https://bonsai-rx.org/docs/articles/subjects.html#behaviorsubject) can be referenced, and written to, from anywhere in your workflow by simply [Multicasting](https://bonsai-rx.org/docs/articles/subjects.html#multicastsubject) a `Harp Message` to it:

:::workflow
![DevicePatternMulticast](~/workflows/device-pattern-multicast.bonsai)
:::

> [!Note]
> We tend to favor `BehaviorSubject` over `PublishSubject` since the `Device` node will terminate the connection with the `Harp Device` if it receives a `OnComplete` event. `BehaviorSubjects` always discard this event, and therefore, will never terminate the connection with the `Harp Device` while your workflow is running.