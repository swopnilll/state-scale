# React Native + WatermelonDB State Anti-patterns

## Architecture Context

This guide addresses state management anti-patterns specific to React Native messaging apps using:
- **WatermelonDB** for local storage with reactive observables
- **Phoenix Channels** for real-time backend communication
- **React Native** for cross-platform mobile development

## Core Concepts

### Observable-Derived State Anti-patterns

- **Rule**: WatermelonDB observables are already reactive - don't duplicate in React state
- **Anti-pattern**: Using `useState` + `useEffect` to sync with WatermelonDB observables
- **Best practice**: Use `withObservables` HOC or direct observable subscription
- **Benefits**:
  - Eliminates double reactivity layers
  - Database is single source of truth
  - Automatic updates when data changes
  - Better performance with fewer re-renders

**Before (Anti-pattern):**

```tsx
function MessageList({ conversationId }: { conversationId: string }) {
  const [messages, setMessages] = useState<Message[]>([]); // ❌ Duplicates database state
  const [unreadCount, setUnreadCount] = useState(0); // ❌ Derived from messages

  useEffect(() => {
    const subscription = database.collections
      .get<Message>('messages')
      .query(Q.where('conversation_id', conversationId))
      .observe()
      .subscribe((messages) => {
        setMessages(messages); // ❌ Syncing observable to state
        setUnreadCount(messages.filter(m => !m.isRead).length); // ❌ Derived calculation in effect
      });

    return () => subscription.unsubscribe();
  }, [conversationId]);

  return (
    <View>
      <Text>Unread: {unreadCount}</Text>
      {messages.map(msg => <MessageItem key={msg.id} message={msg} />)}
    </View>
  );
}
```

**After (Best practice):**

```tsx
const enhance = withObservables(['conversationId'], ({ conversationId }) => ({
  messages: database.collections
    .get<Message>('messages')
    .query(Q.where('conversation_id', conversationId))
    .observe()
}));

function MessageListComponent({ messages }: { messages: Message[] }) {
  // ✅ Derive unread count directly from observable data
  const unreadCount = useMemo(
    () => messages.filter(m => !m.isRead).length,
    [messages]
  );

  return (
    <View>
      <Text>Unread: {unreadCount}</Text>
      {messages.map(msg => <MessageItem key={msg.id} message={msg} />)}
    </View>
  );
}

const MessageList = enhance(MessageListComponent); // ✅ Observable integration
```

### Channel State vs Database State

- **Rule**: Channel events should update database, not component state
- **Anti-pattern**: Storing real-time data in component state parallel to database
- **Best practice**: Phoenix Channel events write to WatermelonDB, components observe database
- **Benefits**:
  - Consistent data across app
  - Persistence survives component unmounts
  - Offline-first architecture
  - Simpler debugging with single data flow

**Before (Anti-pattern):**

```tsx
function ConversationScreen({ conversationId }: { conversationId: string }) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [typingUsers, setTypingUsers] = useState<string[]>([]); // ❌ Channel state in component
  const [onlineStatus, setOnlineStatus] = useState<Record<string, boolean>>({}); // ❌ Ephemeral state mixed with persistent

  useEffect(() => {
    const channel = socket.channel(`conversation:${conversationId}`);
    
    channel.on('new_message', (message) => {
      // ❌ Updates both database AND state
      database.write(async () => {
        await database.collections.get<Message>('messages').create(msg => {
          Object.assign(msg, message);
        });
      });
      setMessages(prev => [...prev, message]); // ❌ Redundant update
    });

    channel.on('typing_start', ({ user_id }) => {
      setTypingUsers(prev => [...prev, user_id]); // ❌ Ephemeral state in component
    });

    channel.on('user_online', ({ user_id, online }) => {
      setOnlineStatus(prev => ({ ...prev, [user_id]: online })); // ❌ Mixed concerns
    });

    return () => channel.leave();
  }, [conversationId]);

  return (
    <View>
      {messages.map(msg => <MessageItem key={msg.id} message={msg} />)}
      <TypingIndicator users={typingUsers} />
    </View>
  );
}
```

**After (Best practice):**

```tsx
// ✅ Separate ephemeral state management
function useChannelState(conversationId: string) {
  const [typingUsers, setTypingUsers] = useState<string[]>([]);
  const [onlineStatus, setOnlineStatus] = useState<Record<string, boolean>>({});

  useEffect(() => {
    const channel = socket.channel(`conversation:${conversationId}`);
    
    channel.on('new_message', (message) => {
      // ✅ Only update database - components will react via observables
      database.write(async () => {
        await database.collections.get<Message>('messages').create(msg => {
          Object.assign(msg, message);
        });
      });
    });

    channel.on('typing_start', ({ user_id }) => {
      setTypingUsers(prev => [...prev, user_id]); // ✅ Appropriate for ephemeral state
    });

    channel.on('typing_stop', ({ user_id }) => {
      setTypingUsers(prev => prev.filter(id => id !== user_id));
    });

    channel.on('user_online', ({ user_id, online }) => {
      setOnlineStatus(prev => ({ ...prev, [user_id]: online })); // ✅ Separated ephemeral state
    });

    return () => channel.leave();
  }, [conversationId]);

  return { typingUsers, onlineStatus };
}

const enhance = withObservables(['conversationId'], ({ conversationId }) => ({
  messages: database.collections
    .get<Message>('messages')
    .query(Q.where('conversation_id', conversationId))
    .observe()
}));

function ConversationScreenComponent({ 
  conversationId, 
  messages 
}: { 
  conversationId: string;
  messages: Message[];
}) {
  const { typingUsers, onlineStatus } = useChannelState(conversationId);

  return (
    <View>
      {messages.map(msg => <MessageItem key={msg.id} message={msg} />)}
      <TypingIndicator users={typingUsers} />
    </View>
  );
}

const ConversationScreen = enhance(ConversationScreenComponent);
```

### Navigation State Anti-patterns

- **Rule**: Don't store navigation-dependent data in component state
- **Anti-pattern**: Loading data in component state that should persist across navigation
- **Best practice**: Use database queries with proper cleanup
- **Mobile-specific concerns**:
  - Background/foreground transitions
  - Memory pressure causing component unmounts
  - Navigation stack changes

**Before (Anti-pattern):**

```tsx
function ContactsList() {
  const [contacts, setContacts] = useState<Contact[]>([]); // ❌ Lost on navigation
  const [searchQuery, setSearchQuery] = useState(''); // ❌ Lost when user navigates away
  const [filteredContacts, setFilteredContacts] = useState<Contact[]>([]); // ❌ Derived state

  useEffect(() => {
    // ❌ Manual data loading that doesn't persist
    database.collections.get<Contact>('contacts').query().fetch()
      .then(setContacts);
  }, []);

  useEffect(() => {
    // ❌ Sync effect for derived data
    setFilteredContacts(
      contacts.filter(c => 
        c.name.toLowerCase().includes(searchQuery.toLowerCase())
      )
    );
  }, [contacts, searchQuery]);

  return (
    <View>
      <TextInput 
        value={searchQuery}
        onChangeText={setSearchQuery}
        placeholder="Search contacts..."
      />
      <FlatList
        data={filteredContacts}
        renderItem={({ item }) => <ContactItem contact={item} />}
      />
    </View>
  );
}
```

**After (Best practice):**

```tsx
// ✅ Global search state that persists across navigation
const searchStore = create<{
  contactsSearchQuery: string;
  setContactsSearchQuery: (query: string) => void;
}>((set) => ({
  contactsSearchQuery: '',
  setContactsSearchQuery: (query) => set({ contactsSearchQuery: query }),
}));

const enhance = withObservables(['searchQuery'], ({ searchQuery }) => ({
  contacts: database.collections
    .get<Contact>('contacts')
    .query(
      searchQuery 
        ? Q.where('name', Q.like(`%${Q.sanitizeLikeString(searchQuery)}%`))
        : Q.where('id', Q.notEq('')) // All contacts
    )
    .observe()
}));

function ContactsListComponent({ contacts }: { contacts: Contact[] }) {
  const { contactsSearchQuery, setContactsSearchQuery } = searchStore();

  return (
    <View>
      <TextInput 
        value={contactsSearchQuery}
        onChangeText={setContactsSearchQuery}
        placeholder="Search contacts..."
      />
      <FlatList
        data={contacts} // ✅ Already filtered by database query
        renderItem={({ item }) => <ContactItem contact={item} />}
      />
    </View>
  );
}

const ContactsList = enhance(ContactsListComponent);
```

### Performance Anti-patterns with Observables

- **Rule**: Be selective about what data you observe
- **Anti-pattern**: Observing entire collections when you only need specific records
- **Best practice**: Use precise queries and avoid over-subscribing
- **Mobile performance concerns**:
  - Battery drain from excessive re-renders
  - Memory usage from large observable subscriptions
  - UI thread blocking from complex calculations

**Before (Anti-pattern):**

```tsx
function MessageBadge({ userId }: { userId: string }) {
  const [allMessages, setAllMessages] = useState<Message[]>([]); // ❌ Observing ALL messages
  const [unreadCount, setUnreadCount] = useState(0);

  useEffect(() => {
    // ❌ Subscribing to all messages when we only need unread count for specific user
    const subscription = database.collections
      .get<Message>('messages')
      .query()
      .observe()
      .subscribe((messages) => {
        setAllMessages(messages);
        const userUnread = messages.filter(m => 
          m.recipientId === userId && !m.isRead
        ).length;
        setUnreadCount(userUnread); // ❌ Heavy calculation on every message change
      });

    return () => subscription.unsubscribe();
  }, [userId]);

  return unreadCount > 0 ? <Badge count={unreadCount} /> : null;
}
```

**After (Best practice):**

```tsx
const enhance = withObservables(['userId'], ({ userId }) => ({
  unreadCount: database.collections
    .get<Message>('messages')
    .query(
      Q.where('recipient_id', userId),
      Q.where('is_read', false)
    )
    .observeCount() // ✅ Only observe count, not full records
}));

function MessageBadgeComponent({ unreadCount }: { unreadCount: number }) {
  return unreadCount > 0 ? <Badge count={unreadCount} /> : null;
}

const MessageBadge = enhance(MessageBadgeComponent); // ✅ Precise, efficient observable
```

### Refs for Non-Reactive Values

- **Rule**: Use refs for values that don't trigger database changes or re-renders
- **Mobile-specific use cases**:
  - Animation values and timers
  - Keyboard and scroll positions
  - Network request cancellation tokens
  - Background task identifiers

**Before (Anti-pattern):**

```tsx
function MessageInput({ conversationId }: { conversationId: string }) {
  const [message, setMessage] = useState('');
  const [typingTimeout, setTypingTimeout] = useState<NodeJS.Timeout | null>(null); // ❌ Causes re-renders
  const [isKeyboardVisible, setIsKeyboardVisible] = useState(false); // ❌ UI state causing re-renders

  const sendTypingEvent = useCallback(() => {
    if (typingTimeout) {
      clearTimeout(typingTimeout); // ❌ Access to stale state
    }
    
    socket.channel(`conversation:${conversationId}`)
      .push('typing_start', { user_id: currentUserId });
    
    const timeout = setTimeout(() => {
      socket.channel(`conversation:${conversationId}`)
        .push('typing_stop', { user_id: currentUserId });
    }, 2000);
    
    setTypingTimeout(timeout); // ❌ Re-render just to store timeout ID
  }, [conversationId, typingTimeout]);

  useEffect(() => {
    const keyboardWillShow = Keyboard.addListener('keyboardWillShow', () => {
      setIsKeyboardVisible(true); // ❌ Re-render for keyboard state
    });
    
    const keyboardWillHide = Keyboard.addListener('keyboardWillHide', () => {
      setIsKeyboardVisible(false);
    });

    return () => {
      keyboardWillShow.remove();
      keyboardWillHide.remove();
    };
  }, []);

  return (
    <View style={[styles.container, { marginBottom: isKeyboardVisible ? 0 : 20 }]}>
      <TextInput
        value={message}
        onChangeText={(text) => {
          setMessage(text);
          sendTypingEvent(); // ❌ Called with stale closure
        }}
      />
    </View>
  );
}
```

**After (Best practice):**

```tsx
function MessageInput({ conversationId }: { conversationId: string }) {
  const [message, setMessage] = useState('');
  const typingTimeoutRef = useRef<NodeJS.Timeout | null>(null); // ✅ No re-renders
  const keyboardOffsetRef = useRef(new Animated.Value(0)); // ✅ Animation values in refs
  const channelRef = useRef<Channel | null>(null);

  useEffect(() => {
    channelRef.current = socket.channel(`conversation:${conversationId}`);
    return () => channelRef.current?.leave();
  }, [conversationId]);

  const sendTypingEvent = useCallback(() => {
    if (typingTimeoutRef.current) {
      clearTimeout(typingTimeoutRef.current); // ✅ Always current value
    }
    
    channelRef.current?.push('typing_start', { user_id: currentUserId });
    
    typingTimeoutRef.current = setTimeout(() => {
      channelRef.current?.push('typing_stop', { user_id: currentUserId });
    }, 2000); // ✅ No re-render when storing timeout
  }, []);

  useEffect(() => {
    const keyboardWillShow = Keyboard.addListener('keyboardWillShow', (e) => {
      Animated.timing(keyboardOffsetRef.current, {
        toValue: e.endCoordinates.height,
        duration: e.duration,
        useNativeDriver: false,
      }).start(); // ✅ Direct animation without state updates
    });
    
    const keyboardWillHide = Keyboard.addListener('keyboardWillHide', (e) => {
      Animated.timing(keyboardOffsetRef.current, {
        toValue: 0,
        duration: e.duration,
        useNativeDriver: false,
      }).start();
    });

    return () => {
      keyboardWillShow.remove();
      keyboardWillHide.remove();
    };
  }, []);

  return (
    <Animated.View 
      style={[
        styles.container, 
        { marginBottom: keyboardOffsetRef.current }
      ]}
    >
      <TextInput
        value={message}
        onChangeText={(text) => {
          setMessage(text);
          sendTypingEvent(); // ✅ Stable reference
        }}
      />
    </Animated.View>
  );
}
```

## Exercise: Fix Mobile Messaging App Anti-patterns

**Scenario**: You're working on a React Native messaging app with the following architecture:
- WatermelonDB for local data storage
- Phoenix Channels for real-time communication
- Navigation between conversation list and individual conversations

### Tasks:

1. **Refactor Observable Integration**
   - Remove useState/useEffect patterns that duplicate WatermelonDB observables
   - Use `withObservables` HOC properly
   - Eliminate derived state calculations in effects

2. **Separate Persistent vs Ephemeral State**
   - Identify what should be stored in database vs component state
   - Ensure Channel updates write to database, not component state
   - Handle typing indicators and presence status appropriately

3. **Optimize Performance**
   - Replace broad observable subscriptions with precise queries
   - Use `observeCount()` instead of observing full collections when possible
   - Move non-reactive values to useRef

4. **Handle Mobile-Specific Concerns**
   - Ensure data persists across navigation and background/foreground transitions
   - Optimize for battery and memory usage
   - Handle keyboard and scroll position without unnecessary re-renders

### Success Criteria:

- Database is single source of truth for persistent data
- No redundant state synchronization between observables and useState
- Proper separation of persistent (database) and ephemeral (component) state
- Efficient observable subscriptions that don't over-subscribe
- Mobile-optimized performance with minimal unnecessary re-renders
