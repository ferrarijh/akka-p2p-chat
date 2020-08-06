# Akka CLI Chat App(p2p)
Simple p2p chat-app using akka pubsub.

## Packages
'''kotlin
import akka.actor.AbstractActor
import akka.actor.ActorRef
import akka.cluster.pubsub.DistributedPubSub
import akka.cluster.pubsub.DistributedPubSubMediator
import mu.KLogging
import java.io.Serializable
'''

## class AbstractActorKL
'''kotlin
abstract class AbstractActorKL: AbstractActor(){
    companion object: KLogging()
}
'''
AbstractActorKL class defined just for convenient implementation of 'logger' in KLogging package.
Other classes will inherit AbstractActorKL instead of AbstractActor.

## Custom messages
'''kotlin
class UserMessage(val from: String, val content: String): Serializable
class ConnectAck(val name: String): Serializable
class Bye(val name: String): Serializable
'''

## User actor
'''kotlin
class User(private var name: String): AbstractActorKL(){  //User Actor sends & displays message
    private val connected = mutableMapOf<String, Boolean>()
    private val path: String = "/user/destination"
    private val mediator: ActorRef = DistributedPubSub.get(context.system).mediator()
    init {
        mediator.tell(DistributedPubSubMediator.Put(self), self)
    }

    override fun preStart() {
        super.preStart()
        mediator.tell(DistributedPubSubMediator.SendToAll(path, ConnectAck(name), true), self)
    }

    override fun createReceive(): Receive = receiveBuilder()
        .match(ConnectAck::class.java) {
            if (connected[it.name] != true) {
                connected[it.name] = true
                println("\r>> new user '${it.name}' connected!")
                print(">> [me]")
            }
        }.match(String::class.java){
            if (it=="bye"){
                mediator.tell(DistributedPubSubMediator.SendToAll(path, Bye(name), true), self)
                print(">> See you again!")
                context.system.terminate()
            }else {
                val msg = UserMessage(name, it)
                mediator.tell(DistributedPubSubMediator.SendToAll(path, msg, true), self)
            }
        }.match(UserMessage::class.java) {
            println("\r>> [${it.from}] ${it.content}")
            print(">> [me]")
        }.match(Bye::class.java){
            println("\r>> user '${it.name}' disconnected.")
            print(">> [me]")
            connected[it.name] = false
        }.build()
}
'''
