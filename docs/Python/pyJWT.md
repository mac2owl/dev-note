[PyJWT](https://pyjwt.readthedocs.io/en/latest/)

## Validate and return user with AWS cognito access token

```py
import json

from typing import Optional

import jwt
import requests

from jwt.algorithms import RSAAlgorithm

from path.to.user_model import User
from path.to.config import config


class UserAuthService:
    def get_current_user(self, access_token: str = None) -> Optional[User]:
        valid_token, current_user = self.validate_and_return_credentials(access_token)
        return current_user if (valid_token and current_user) else None

    def get_public_access_keys(self) -> str:
        auth = config.auth
        url = f"https://cognito-idp.{auth.region}.amazonaws.com/{auth.identity_pool_id}/.well-known/jwks.json"
        response = requests.get(url)
        return response.text

    def find_public_key(self, kid: str) -> tuple[bool, Optional[dict]]:
        public_keys = self.get_public_access_keys()
        public_keys = json.loads(public_keys)
        public_keys = {key["kid"]: key for key in public_keys["keys"]}
        matched_key = public_keys.get(kid)
        return bool(matched_key), matched_key

    def decode_access_token(self, public: dict, access_token: str, alg: str) -> tuple[bool, dict]:
        try:
            public_key = RSAAlgorithm.from_jwk(json.dumps(public))
            payload = jwt.decode(
                access_token,
                public_key,
                algorithms=[alg],
                verify=True,
                options={
                    "verify_exp": True,
                    "verify_nbf": True,
                    "verify_iat": False, # True
                    "verify_aud": False, # True
                    "verify_iss": True,
                },
                audience=config.auth.identity_pool_web_client,
                issuer=f"https://cognito-idp.{config.auth.region}.amazonaws.com/{config.auth.identity_pool_id}",
            )
        except jwt.exceptions.DecodeError as exc:
            return False, {}

        # This should be check if token_use is ID token and "verify_aud" in jwt decode
        if payload["token_use"] != "access":
            return False, {}

        if payload["client_id"] != config.auth.identity_pool_web_client:
            return False, {}

        return True, payload

    def validate_and_return_credentials(self, access_token: str) -> tuple[bool, Optional[User]]:
        valid_token, current_user = False, None

        try:
            headers = jwt.get_unverified_header(access_token)
            valid_pub_key, user_public_key = self.find_public_key(headers.get("kid"))
            if not user_public_key:
                return False, None

            valid_signature, payload = self.decode_access_token(user_public_key, access_token, headers.get("alg"))
            if valid_pub_key and valid_signature:
                valid_token = True
                user_sub_id = payload.get("sub")
                current_user = User.find_by(guid=user_sub_id) if user_sub_id else None
        except Exception as exc:
						return False, None

        return valid_token, current_user

```

### With ALB token verification ([ALB Beast vulnerability](https://thehackernews.com/2024/08/new-albeast-vulnerability-exposes.html))

```python
import json

from typing import Optional, Tuple

import jwt
import requests

from jwt.algorithms import RSAAlgorithm

from app.config import config
from path.to.user_model import User


class UserAuthService:
    def get_current_user(self, access_token: str = None, data_token: str = None):
        is_valid_alb_token = self.verify_alb_token(data_token)
        if not is_valid_alb_token:
            return None

        valid_token, current_user = self.validate_access_token_and_return_credentials(
            access_token
        )
        return current_user if (valid_token and current_user) else None


    def get_public_access_keys(self, url: str) -> Optional[str]:
        try:
            response = requests.get(url)
            response.raise_for_status()
            return response.text
        except Exception as exc:
            return None

    def find_public_key(self, kid: str) -> Tuple[bool, Optional[dict]]:
        auth = config.auth
        url = f"https://cognito-idp.{auth.region}.amazonaws.com/{auth.identity_pool_id}/.well-known/jwks.json"
        public_keys = self.get_public_access_keys(url)
        if not public_keys:
            return False, {}

        public_keys = json.loads(public_keys)
        public_keys = {key["kid"]: key for key in public_keys["keys"]}
        matched_key = public_keys.get(kid)
        return bool(matched_key), matched_key

    def decode_access_token(self, public: dict, access_token: str, alg: str) -> tuple[bool, dict]:
        try:
            public_key = RSAAlgorithm.from_jwk(json.dumps(public))
            payload = jwt.decode(
                access_token,
                public_key,
                algorithms=[alg],
                verify=True,
                options={
                    "verify_exp": True,
                    "verify_nbf": True,
                    "verify_iat": False,
                    "verify_aud": False,
                    "verify_iss": True,
                },
                audience=config.auth.identity_pool_web_client,
                issuer=f"https://cognito-idp.{config.auth.region}.amazonaws.com/{config.auth.identity_pool_id}",
            )
        except jwt.exceptions.DecodeError as exc:
            return False, {}

        # This should be check if token_use is ID token and "verify_aud" in jwt decode
        if payload["token_use"] != "access":
            return False, {}

        if payload["client_id"] != config.auth.identity_pool_web_client:
            return False, {}

        return True, payload

    def verify_alb_token(self, data_token: str) -> bool:
        if not data_token:
            return False

        headers = jwt.get_unverified_header(data_token)
        auth = config.auth
        expected_signer = (
            f"arn:aws:elasticloadbalancing:{auth.region}:{auth.alb_id}:"
            f"loadbalancer/app/{auth.load_balancer_name}/{auth.load_balancer_id}"
        )
        if not headers.get("signer") == expected_signer:
            return False

        url = f'https://public-keys.auth.elb.{auth.region}.amazonaws.com/{headers.get("kid")}'
        public_key = self.get_public_access_keys(url)
        if not public_key:
            return False

        expected_issuer = f"https://cognito-idp.{config.auth.region}.amazonaws.com/{config.auth.identity_pool_id}"
        payload = jwt.decode(
            data_token,
            public_key,
            algorithms=[headers["alg"]],
            verify=True,
            options={
                "verify_exp": True,
                "verify_nbf": True,
                "verify_iat": False,
                "verify_aud": False,
                "verify_iss": True,
            },
            audience=config.auth.identity_pool_web_client,
            issuer=expected_issuer,
        )

        return payload.get("iss") == expected_issuer

    def validate_access_token_and_return_credentials(self, access_token: str) -> Tuple[bool, Optional[str], bool]:
        valid_token, user_sub_id, current_user = False, None, None
        try:
            headers = jwt.get_unverified_header(access_token)
            valid_pub_key, user_public_key = self.find_public_key(headers.get("kid"))
            if not user_public_key:
                return False, None, None

            valid_signature, payload = self.decode_access_token(user_public_key, access_token, headers.get("alg"))
            if valid_pub_key and valid_signature:
                valid_token = True
                user_sub_id = payload.get("sub")
                current_user = User.find_by(guid=user_sub_id) if user_sub_id else None
        except Exception as exc:
            pass

        return valid_token, current_user
```
