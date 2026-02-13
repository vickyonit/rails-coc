---

# 14 — Action Cable Conventions

---

## 14.1 Channel Structure

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private

    def find_verified_user
      if verified_user = User.find_by(id: cookies.encrypted[:user_id])
        verified_user
      else
        reject_unauthorized_connection
      end
    end
  end
end

# app/channels/application_cable/channel.rb
module ApplicationCable
  class Channel < ActionCable::Channel::Base
  end
end
```

```ruby
# app/channels/chat_channel.rb
class ChatChannel < ApplicationCable::Channel
  def subscribed
    @room = Room.find(params[:room_id])
    stream_for @room
    # or: stream_from "chat_#{params[:room_id]}"
  end

  def unsubscribed
    # Cleanup
  end

  def speak(data)
    message = @room.messages.create!(
      user: current_user,
      body: data["message"]
    )
    ChatChannel.broadcast_to(@room, {
      message: render_message(message)
    })
  end

  private

  def render_message(message)
    ApplicationController.render(
      partial: "messages/message",
      locals: { message: message }
    )
  end
end
```

---

## 14.2 Broadcasting Conventions

```ruby
# From anywhere (model, job, service):
ChatChannel.broadcast_to(room, { message: "Hello!" })
ActionCable.server.broadcast("chat_#{room.id}", { message: "Hello!" })

# Convention: Broadcast from models via callbacks or jobs
class Message < ApplicationRecord
  after_create_commit :broadcast_message

  private

  def broadcast_message
    broadcast_append_to room, target: "messages"
    # Uses Turbo Streams (Rails 7+)
  end
end
```