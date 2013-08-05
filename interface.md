Interface 
=================

### Company Account Sign in

request: 

    POST /api/company_signin
    data : {
        company     : "",           //string, company name
        username    : "",           //string, user name
        password    : ""            //string, password encoded in md5
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : ""            //string, error description
    }


### Add User

request: 

    POST /api/company_user_add
    data : {
        company     : "",           //string, company name
        username    : "",           //string, user name
        password    : "",           //string, default password encoded in md5
        attributes  : {
            email       : "",
            telephone   : "",
            mobile      : "",
            ...,                    // any attributes we want
            description : "",
        },        
        roles : [
            "admin", 
            "user", 
            ...
        ]                           //array of string, roles assign to this user
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        user_id     : 0             //int, user id
    }


### Remove User

request: 

    POST /api/company_user_rmv
    data : {
        user_id     : 0,            //int, user id, key to delete a user
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : ""            //string, error description
    }

### Edit User

request: 

    POST /api/company_user_edit
    data : {
        user_id     : 0,            //int, user id, key to edit a user's profile
        attributes  : {
            email       : "",
            telephone   : "",
            mobile      : "",
            ...,                    // any attributes we want
            description : "",
        },        
        roles : [
            "admin", 
            "user", 
            ...
        ]                           //array of string, roles assign to this user
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
    }


### List Users

request: 

    POST /api/company_user_list
    data : {
        filter : {
            company     : "",       //string, company name
            ...                     //maybe some other filters
        },
        page_no     : 0,            //current page number
        page_step   : 0,            //how many items you want to show per page
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        page_no     : 0,            //current page number
        prev_page   : 0,            //prev page no
        next_page   : 0,            //next page no
        last_page   : 0,            //last page no
        user_profiles: [
            {
                user_id     : 0,            //int, user id, key to edit a user's profile
                company     : "",           //string, company name
                username    : "",           //string, user name
                attributes  : {
                    email       : "",
                    telephone   : "",
                    mobile      : "",
                    ...,                    // any attributes we want
                    description : "",
                },        
                roles : [
                    "admin", 
                    "user", 
                    ...
                ]                           //array of string, roles assign to this user
            },
            ...
        ]
    }

### Get Users Count

request: 

    POST /api/company_user_count
    data : {
        filter : {
            company     : "",       //string, company name
            ...                     //maybe some other filters
        }
    }

response: 

    data : {
        ret_code    : 0,            //int, 0 for success; other for failure
        err_str     : "",           //string, error description
        user_count  : 0,            //int, query count
    }


