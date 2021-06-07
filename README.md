- 👋 Hi, I’m @skinwalker11
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
from telegram import (User, Message, Update, Chat, ChatMember, UserProfilePhotos, File,
                      ReplyMarkup, TelegramObject, WebhookInfo, GameHighScore, StickerSet,
                      PhotoSize, Audio, Document, Sticker, Video, Animation, Voice, VideoNote,
                      Location, Venue, Contact, InputFile, Poll, BotCommand)
from telegram.error import InvalidToken, TelegramError
from telegram.utils.helpers import to_timestamp, DEFAULT_NONE
                      Location, Venue, Contact, InputFile, Poll, InlineKeyboardMarkup, BotCommand)
from telegram.error import InvalidToken, TelegramError, InvalidCallbackData
from telegram.utils.helpers import to_timestamp, DEFAULT_NONE, sign_callback_data
from telegram.utils.request import Request

logging.getLogger(__name__).addHandler(logging.NullHandler())
@@ -88,6 +88,9 @@ class Bot(TelegramObject):
        private_key_password (:obj:`bytes`, optional): Password for above private key.
        defaults (:class:`telegram.ext.Defaults`, optional): An object containing default values to
            be used if not set explicitly in the bot methods.
        validate_callback_data (:obj:`bool`, optional): Whether the callback data of
            :class:`telegram.CallbackQuery` updates recieved by this bot should be validated. For
            more info, please see our wiki. Defaults to :obj:`True`.
    """

@@ -129,12 +132,17 @@ def __init__(self,
                 request=None,
                 private_key=None,
                 private_key_password=None,
                 defaults=None):
                 defaults=None,
                 validate_callback_data=True):
        self.token = self._validate_token(token)

        # Gather default
        self.defaults = defaults

        # Dictionary for callback_data
        self.callback_data = {}
        self.validate_callback_data = validate_callback_data

        if base_url is None:
            base_url = 'https://api.telegram.org/bot'

@@ -155,6 +163,15 @@ def __init__(self,

    def _message(self, url, data, reply_to_message_id=None, disable_notification=None,
                 reply_markup=None, timeout=None, **kwargs):
        def _replace_callback_data(reply_markup, chat_id):
            if isinstance(reply_markup, InlineKeyboardMarkup):
                for button in [b for l in reply_markup.inline_keyboard for b in l]:
                    if button.callback_data:
                        self.callback_data[str(id(button.callback_data))] = button.callback_data
                        button.callback_data = sign_callback_data(chat_id,
                                                                  str(id(button.callback_data)),
                                                                  self)

        if reply_to_message_id is not None:
            data['reply_to_message_id'] = reply_to_message_id

@@ -163,6 +180,8 @@ def _message(self, url, data, reply_to_message_id=None, disable_notification=Non

        if reply_markup is not None:
            if isinstance(reply_markup, ReplyMarkup):
                # Replace callback data by their signed id
                _replace_callback_data(reply_markup, data['chat_id'])
                # We need to_json() instead of to_dict() here, because reply_markups may be
                # attached to media messages, which aren't json dumped by utils.request
                data['reply_markup'] = reply_markup.to_json()
@@ -2108,9 +2127,12 @@ def get_updates(self,
            2. In order to avoid getting duplicate updates, recalculate offset after each
               server response.
            3. To take full advantage of this library take a look at :class:`telegram.ext.Updater`
            4. The renutred list may contain :class:`telegram.error.InvalidCallbackData` instances.
               Make sure to ignore the corresponding update id. For more information, please see
               our wiki.
        Returns:
            List[:class:`telegram.Update`]
            List[:class:`telegram.Update` | :class:`telegram.error.InvalidCallbackData`]
        Raises:
            :class:`telegram.TelegramError`
@@ -2144,7 +2166,15 @@ def get_updates(self,
            for u in result:
                u['default_quote'] = self.defaults.quote

        return [Update.de_json(u, self) for u in result]
        updates = []
        for u in result:
            try:
                updates.append(Update.de_json(u, self))
            except InvalidCallbackData as e:
                e.update_id = int(u['update_id'])
                self.logger.warning('{} Malicious update: {}'.format(e, u))
                updates.append(e)
        return updates

    @log
    def set_webhook(self,
  9  telegram/callbackquery.py 
@@ -19,6 +19,7 @@
"""This module contains an object that represents a Telegram CallbackQuery"""

from telegram import TelegramObject, Message, User
from telegram.utils.helpers import validate_callback_data


class CallbackQuery(TelegramObject):
@@ -107,6 +108,14 @@ def de_json(cls, data, bot):
            message['default_quote'] = data.get('default_quote')
        data['message'] = Message.de_json(message, bot)

        if bot is not None:
            chat_id = data['message'].chat.id
            if bot.validate_callback_data:
                key = validate_callback_data(chat_id, data['data'], bot)
            else:
                key = validate_callback_data(chat_id, data['data'])
            data['data'] = bot.callback_data.get(key, None)

        return cls(bot=bot, **data)

    def answer(self, *args, **kwargs):
  13  telegram/error.py 
@@ -111,3 +111,16 @@ class Conflict(TelegramError):

    def __init__(self, msg):
        super(Conflict, self).__init__(msg)


class InvalidCallbackData(TelegramError):
    """
    Raised when the received callback data has been tempered with.
    Args:
        update_id (:obj:`int`, optional): The ID of the untrusted Update.
    """
    def __init__(self, update_id=None):
        super(InvalidCallbackData, self).__init__('The callback data has been tampered with! '
                                                  'Skipping it.')
        self.update_id = update_id
  31  telegram/ext/basepersistence.py 
@@ -33,6 +33,8 @@ class BasePersistence(object):
      :meth:`update_user_data`.
    * If you want to store conversation data with :class:`telegram.ext.ConversationHandler`, you
      must overwrite :meth:`get_conversations` and :meth:`update_conversation`.
    * If :attr:`store_callback_data` is :obj:`True`, you must overwrite :meth:`get_callback_data`
      and :meth:`update_callback_data`.
    * :meth:`flush` will be called when the bot is shutdown.
    Attributes:
@@ -42,6 +44,8 @@ class BasePersistence(object):
            persistence class.
        store_bot_data (:obj:`bool`): Optional. Whether bot_data should be saved by this
            persistence class.
        store_callback_data (:obj:`bool`): Optional. Whether callback_data be saved by this
            persistence class.
    Args:
        store_user_data (:obj:`bool`, optional): Whether user_data should be saved by this
@@ -50,12 +54,16 @@ class BasePersistence(object):
            persistence class. Default is ``True`` .
        store_bot_data (:obj:`bool`, optional): Whether bot_data should be saved by this
            persistence class. Default is ``True`` .
        store_callback_data (:obj:`bool`, optional): Whether callback_data should be saved by this
            persistence class. Default is ``True`` .
    """

    def __init__(self, store_user_data=True, store_chat_data=True, store_bot_data=True):
    def __init__(self, store_user_data=True, store_chat_data=True, store_bot_data=True,
                 store_callback_data=True):
        self.store_user_data = store_user_data
        self.store_chat_data = store_chat_data
        self.store_bot_data = store_bot_data
        self.store_callback_data = store_callback_data

    def get_user_data(self):
        """"Will be called by :class:`telegram.ext.Dispatcher` upon creation with a
@@ -83,7 +91,17 @@ def get_bot_data(self):
        ``dict``.
        Returns:
            :obj:`defaultdict`: The restored bot data.
            :obj:`dict`: The restored bot data.
        """
        raise NotImplementedError

    def get_callback_data(self):
        """"Will be called by :class:`telegram.ext.Dispatcher` upon creation with a
        persistence object. It should return the callback_data if stored, or an empty
        ``dict``.
        Returns:
            :obj:`dict`: The restored bot data.
        """
        raise NotImplementedError



